---
layout: post
title: "1号进程（init）信号处理的特殊处理逻辑"
category: GNU-linux
date: 2023-03-28 08:00:00 +0800
---

1号进程（init进程）在linux的信号处理流程中有特殊的处理逻辑。例如我们向普通的进程发送SIGKILL，进程会退出；向1号进程发送SIGKILL则好像无事发生过。

```
[root@openEuler-2203 test]# sleep 1234 &
[1] 2843
[root@openEuler-2203 test]# kill -9 2843
[root@openEuler-2203 test]# ps aux | grep 2843
root        2845  0.0  0.1   6420  2136 pts/0    S+   11:00   0:00 grep --color=auto 2843
[1]+  Killed                  sleep 1234
[root@openEuler-2203 test]# kill -9 1
[root@openEuler-2203 test]# ps aux | grep systemd
root           1  0.0  0.7 165064 11384 ?        Ss   May05   0:02 /usr/lib/systemd/systemd --switched-root --system --deserialize 16
root         622  0.0  0.5  18188  8780 ?        Ss   May05   0:00 /usr/lib/systemd/systemd-journald
root         658  0.0  0.5  28220  8884 ?        Ss   May05   0:00 /usr/lib/systemd/systemd-udevd
root         771  0.0  0.5  14924  7896 ?        Ss   May05   0:00 /usr/lib/systemd/systemd-logind
root        1535  0.0  0.6  16156  9648 ?        Ss   May05   0:00 /usr/lib/systemd/systemd --user
root        2847  0.0  0.1   6420  2172 pts/0    S+   11:00   0:00 grep --color=auto systemd
```

## 1号进程的特殊性

linux有以下几个地方会特殊处理1号进程

1. SIGNAL_UNKILLABLE
2. is_global_init
3. is_child_reaper

### SIGNAL_UNKILLABLE

从SIGNAL_UNKILLABLE定义的注释，我们能清楚地看到，这个标记位是1号进程独享的，其作用就是会忽略发向1号进程的致命信号。

```c
#define SIGNAL_UNKILLABLE	0x00000040 /* for init: ignore fatal signals */
```

通过`is_child_reaper()`函数判断进程为当前名字空间的1号进程时，内核会为该进程的signal->flags新增SIGNAL_UNKILLABLE标记位。

```c
/* kernel/fork.c */
if (is_child_reaper(pid)) {
    ns_of_pid(pid)->child_reaper = p;
    p->signal->flags |= SIGNAL_UNKILLABLE;
}
```

## 如何屏蔽发向1号进程的信号？

`sig_task_ignored()`函数是屏蔽发向1号进程信号的主要逻辑。

```c
/* kernel/signal.c */
static bool sig_task_ignored(struct task_struct *t, int sig, bool force)
{
	void __user *handler;

	handler = sig_handler(t, sig);

	/* SIGKILL and SIGSTOP may not be sent to the global init */
	if (unlikely(is_global_init(t) && sig_kernel_only(sig)))
		return true;

	if (unlikely(t->signal->flags & SIGNAL_UNKILLABLE) &&
	    handler == SIG_DFL && !(force && sig_kernel_only(sig)))
		return true;

	/* Only allow kernel generated signals to this kthread */
	if (unlikely((t->flags & PF_KTHREAD) &&
		     (handler == SIG_KTHREAD_KERNEL) && !force))
		return true;

	return sig_handler_ignored(handler, sig);
}
```

1. 第一个if：如果是1号进程，并且发送的信号为SIGKILL/SIGSTOP，那么信号不会被发送出去。
2. 第二个if：如果这个信号处理函数为缺省的，并且已经设置了SIGNAL_UNKILLABLE标记位，信号也不会发送出去。

根据第一个if判断，1号进程肯定会屏蔽掉SIGKILL/SIGSTOP信号。由于systemd在启动的早期阶段调用`reset_all_signal_handlers()`将所有的信号处理函数设置为SIG_DFL，因此根据第二个if判断，其他信号也不会发送到1号进程。

```c
/* systemd在main函数的最开始会调用reset_all_signal_handlers将所有
 * 信号的信号处理函数修改为SIG_DFL。
 * systemd: src/basic/signal-util.c */
int reset_all_signal_handlers(void) {
        static const struct sigaction sa = {
                .sa_handler = SIG_DFL,
                .sa_flags = SA_RESTART,
        };
        int r = 0;

        for (int sig = 1; sig < _NSIG; sig++) {

                /* These two cannot be caught... */
                if (IN_SET(sig, SIGKILL, SIGSTOP))
                        continue;

                /* On Linux the first two RT signals are reserved by
                 * glibc, and sigaction() will return EINVAL for them. */
                if (sigaction(sig, &sa, NULL) < 0)
                        if (errno != EINVAL && r >= 0)
                                r = -errno;
        }

        return r;
}
```

## systemd还是会响应一些信号？

有经验的读者会注意到，SIGSEGV、SIGBUS、SIGABRT等致命信号会使systemd产生coredump，此外systemd还会响应SIGTERM、SIGHUP等信号分别执行`systemctl daemon-reexec`，`systemctl daemon-reload`。

触发coredump是因为systemd在通过`reset_all_signal_handlers()`重设信号处理函数后，调用`install_crash_handler()`为7种致命信号注册了新的信号处理函数：

```c
/* systemd: src/basic/constant.h */
#define SIGNALS_CRASH_HANDLER SIGSEGV,SIGILL,SIGFPE,SIGBUS,SIGQUIT,SIGABRT

/* systemd: src/core/crash-handler.c */
void install_crash_handler(void) {
        static const struct sigaction sa = {
                .sa_sigaction = crash,
                .sa_flags = SA_NODEFER | SA_SIGINFO, /* So that we can raise the signal again from the signal handler */
        };
        int r;

        /* We ignore the return value here, since, we don't mind if we cannot set up a crash handler */
        r = sigaction_many(&sa, SIGNALS_CRASH_HANDLER);
        if (r < 0)
                log_debug_errno(r, "I had trouble setting up the crash handler, ignoring: %m");
}
```

SIGTERM、SIGHUP等信号则是systemd在`manager_setup_signals()`函数中，通过调用`signalfd()`创建信号监听fd，使用`sd_event_add_io()`监听的。这种监听方法允许systemd在合适的时间处理这些信号，并且能够接受`manager`作为入参做更多的事情。

除此之外，systemd还主动忽略了SIGPIPE信号：

```c
/* systemd将SIG_IGN作为SIGPIPE的信号处理函数。
 * systemd: src/core/constant.h */
#define SIGNALS_IGNORE SIGPIPE
/* systemd: src/core/main.c */
(void) ignore_signals(SIGNALS_IGNORE);
```

这里给出最基础的代码走读逻辑，后续再遇到相关问题时继续完善。
