---
layout: post
title: "性能分析和调优"
category: kernel
date: 2024-09-28 15:00:00 +0800
---

## 参考书

<https://weedge.github.io/perf-book-cn/zh/>

## 通常步骤

1. 业务输出数据流图、性能瓶颈点
2. 业务输出部署视图：有哪些业务进程、如何绑核、运行在虚拟机还是物理机
3. 数据采集
    * top信息
    * syscall
    * 中断
    * 热点函数（进程粒度、VCPU粒度、动态库粒度、函数粒度），生成火焰图
    * topdown信息
4. 针对热点函数和性能瓶颈开展优化

## perf

### 常用参数

* `-a`：所有CPU
* `-C`：指定CPU
* `-p`：指定PID
* `-o`：指定输出文件
* `-g`：使能调用栈，支持调用关系图
* `-F`：采样频率
* `-e`：事件
* `--filter`：event时间过滤

<https://www.cnblogs.com/arnoldlu/p/6241297.html>

### 常用命令

#### 热点函数

* `perf record -g -p PID -- sleep 20`
* `perf top -p PID -g -d 100`（-d表示多久刷新一次数据）

#### 调度、时延

* `perf sched record`（通过sched record采集实验数据，然后需要在使用`perf sched latency`处理数据）

需要注意，使用`porf sched record`采集的数据必须用`perf sched`子命令来解析，而不是使用`perf report`。

#### 事件统计

* `perf stat -e dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses ./test`
* `perf stat -p 1 -e cache-reference -e cache-misses`
* `perf record -e dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses -p PID -o perf.data -- sleep 10` => `perf report -i perf.data`

#### 微架构数据（TopDown）

基础知识：<https://zhuanlan.zhihu.com/p/34688930>
<https://zhuanlan.zhihu.com/p/60569271>

* Frontend bound（前端依赖）首先需要注意的是这里的前端并不是指UI的前端，这里的前端指的是x86指令解码阶段的耗时。
* Backend bound（后端依赖）同样不同于其他“后端”的定义，这里指的是传统的CPU负责处理实际事务的能力。由于这一个部分相对其他部分来说，受程序指令的影响更为突出，这一块又划分出了两个分类。core bound（核心依赖）意味着系统将会更多的依赖于微指令的处理能力。memory bound（存储依赖）我这里不把memory翻译成内存的原因在于这里的memory包含了CPU L1～L3缓存的能力和传统的内存性能。
* Bad speculation（错误的预测）这一部分指的是由于CPU乱序执行预测错误导致额外的系统开销。
* Retiring（拆卸）字面理解是退休的意思，事实上这里指的是指令完成、等待指令切换，模块重新初始化的开销。

【伪共享】多线程的程序可能存在`False sharing`：多个线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会导致缓存行失效和频繁替换。<https://zhuanlan.zhihu.com/p/65394173> <https://www.cnblogs.com/usmiles/p/15627229.html>

`perf state -p 1 -ddd -- sleep 60`

* IPC：每个cycle的Instruction数量（越大越好，最好能达到甚至超过1）
* context-switches：进程切换次数（让出cpu给其他进程使用）
* cpu-migrations：当前进程在不同的CPU之间迁移的次数

### 数据解析

* `perf report --fields=overhead,symbol,dso --column-widths=0,40,0 -i perf.data`
* 火焰图：`perf script > out.perf`
* `perf annotate`：显示源代码的性能相关注释，帮助解决性能问题。(注意：依赖objdump，按下o后能够显示汇编代码位置，O显示汇编代码的相对位置。)

## 硬件相关

### 硬件预取

硬件预取：硬件预取器通过跟踪Load指定数据地址的变化规律来预测将会被访问到的内存地址，并提前从DRAM中读取这些数据到Cache。

* OBL预取器：One Block Loadahead，每次预取下一个数据块到Cache里。
* Stream流预取器：从OBL预取器的基础上发展而来，根据访存地址单项递增或递减导致的连续Cache Miss预取。
* Stride跨步预取器：跟踪和预取的不再是连续的Cache line，可以有固定的跨度。

存在的问题：

* 仅支持Stream和Stride：对毫无规律的随机内存访问硬件无法识别，没有效果。
* 资源限制：需要追踪识别访存流，需要多次Cache Miss才能识别出流的方向。
* 无法识别循环边界：对超越数组边界的无用数据也会发起预取。

## stress-ng

待补充

## 可用于性能分析的内核接口

* `/proc/sched_debug`：详细的调度统计数据，可视化的
* `/proc/schedstat`：一连串数字，可以看到调度域信息
* `/proc/sys/kernel/sched_schedstats`：是否启用kernel debug，启用后可能会有性能下降
* `/proc/interrupts`：查看每个CPU的中断个数以及对应的中断编号

## 参数调优

|参数名|作用|使用场景|
|-|-|-|
|vm.dirty_writeback_centisecs|配置定期回写的间隔<br>0：禁用定期回写<br>|平衡内存使用和IO性能。对内存紧张的，使用低回写间隔，提高内存利用率；对IO性能差的使用高回写间隔，避免回写成为性能瓶颈|
|vm.swappiness|回收匿名页的激进程度（a），越高越倾向于回收匿名页，取值范围：[0,100]<br>匿名页:文件页=a:(200-a)<br>0：禁用匿名页回收<br>|对数据库这种自己对内存做管理的业务，最好将vm.swappiness配置为0，不让内核回收匿名页|
|vm.dirty_background_ratio|脏页数量达到这里设置的百分比时，开始后台脏页回写。|-|

## 优化手段

### 绑核

通过绑核降低任务在不同CPU间迁移的次数，降低任务迁移的开销。与绑核类似的手段还包括：减少空闲负载均衡次数、周期负载均衡次数以及任务唤醒切核次数。

## 中断

减少中断的触发次数，使用低精度计时器。

## cache、TLB miss

使用大页降低TLB miss，通过数据预取（包括硬件预取、软件预取）降低L3 cache miss（也叫LLC cache，Last Level Cache）。

## 编译优化

link时新增编译选项、数据结构对齐。
