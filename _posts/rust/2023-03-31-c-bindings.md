---
layout: post
title: "在rust中调用c"
category: rust
date: 2023-03-28 08:00:00 +0800
---

## 在rust中调用c库

以libselinux的`is_selinux_enabled()`为例说明。调用c库需要做好两件事：

1. 配置好link
2. 做好接口封装

最简代码如下：

```rust
use std::os::raw::c_int;

#[link(name = "selinux")]
extern "C" {
    fn is_selinux_enabled() -> c_int;
}

pub fn selinux_enabled() -> bool {
    let res = unsafe { is_selinux_enabled() };
    res > 0
}
```

这里我们用`extern "C"`声明函数`is_selinux_enabled()`是外部的c函数，并通过`#[link(name = "selinux")]`告诉编译器去链接`libselinux`，类似于`gcc -lselinux`。

`selinux_enabled()`是对`is_selinux_enabled()`再进行一次封装，使得后续调用时不要都带上`unsafe`。

## 使用rust的libc

编码中为了减少依赖，经常会用到rust的libc，调用rust的libc最困难的是如何构造输入，解析输出，怎么做好数据类型的转换。这里列出一些常见类型的初始化方法。

```rust
// *const libc::c_char
let my_char = std::ffi::CString::new("Hello").unwrap();
// 在需要使用`c_char`的地方，使用`my_char.as_ptr()`。

// 结构体全零初始化
let mut pglob: libc::glob_t = unsafe {
    std::mem::zeroed()
};
// 需要使用结构体的指针时：`&mut pglob`
// 结构体直接初始化
let action = sigaction {
    sa_sigaction: handle_sigint as usize,
    sa_mask: 0,
    sa_flags: 0,
    sa_restorer: None,
};
// 空指针NULL
std::ptr::null()和std::ptr::null_mut() //多数场景用的是后者。
```
