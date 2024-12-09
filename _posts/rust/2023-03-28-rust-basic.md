---
layout: post
title: "rust的基础知识"
category: rust
date: 2023-03-28 08:00:00 +0800
---

## 1 trait

<https://zhuanlan.zhihu.com/p/127365605> trait的用法

### 如何将rust的trait转换为对应的struct？

实测可行：

<https://stackoverflow.com/questions/33687447/how-to-get-a-reference-to-a-concrete-type-from-a-trait-object>

## 2 智能指针

### Box

智能指针，指向堆上的数据。通过实现`Deref`来解引用`Box`。

### 计数智能指针和多重所有权

Rc<T>实现引用计数，确保这个智能指针可以在多个方法中使用。

```rust
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a));
    let c = Cons(4, Rc::clone(&a));
}
```

### RefCell

RefCell在运行时检查借用规则，而不是在编译时。内部可变性。

## 3 异步编程

### 3.1 await()的理解

调用await(),意思是等待当前的操作执行完后,再执行下一步操作.但是在等的过程中,当前是可以执行其它任务的.
如果调用thread::sleep()等待,程序会阻塞,不能执行其它任务.

### 3.2 如何同时运行多个future?

join!, try_join!: 在future结束后,集中处理结果.try_join!会在单个future报错时及时中止.
select!:

### 3.3 block_on会非阻塞地等待当前任务处理完成

### 3.4 帮助理解asynchonous runtime

[rust-async-runtime](https://www.ncameron.org/blog/what-is-an-async-runtime/)


## 4 错误码的处理

```rs
#[derive(Debug, thiserror::Error)]
pub enum MyError {
    #[error("ioerror")]
    IOError(#[from] std::io::Error),
    #[error("others")]
    Others,
}

fn return_std_io_error() -> Result<(), std::io::Error> {
    Err(std::io::Error::new(std::io::ErrorKind::InvalidData, "crossesdevices"))
}

fn return_myerror() -> Result<(), MyError> {
    // if let Err(e) = return_std_io_error() {
    //     return Err(e.into());
    // }
    // Ok(())
    return_std_io_error()?;
    Ok(())
}

fn main() {
    if let Err(e) = return_myerror() {
        println!("res: {}", e.to_string());
    }
}
```

借用thiserror简化Error的处理。return_myerror函数中的`?`可以自动进行Error的类型转换，`Ok(())`继续向下走。等价于注视后的代码。

另外注意点：IOError(#[from] std::io::Error)实现了From,那么就不能重复为Others(#[from] std::io::Error)。编译器已经想到了，不允许相同的来源转换成不同的结果。

```rs
use std::error::Error;

#[derive(Debug)]
pub enum MyError {
    IOError(std::io::Error),
    Others,
}

impl std::fmt::Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let res = match self {
            Self::IOError(_) => "ioerror",
            Self::Others => "others",
        };
        write!(f, "{res}")
    }
}

impl Error for MyError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            Self::IOError(e) => Some(e),
            Self::Others => None,
        }
    }
}

fn return_std_io_error() -> Result<(), std::io::Error> {
    Err(std::io::Error::new(std::io::ErrorKind::InvalidData, "crossesdevices"))
}

fn return_myerror() -> Result<(), MyError> {
    if let Err(e) = return_std_io_error() {
        return Err(MyError::IOError(e));
    }
    Ok(())
}

fn main() {
    if let Err(e) = return_myerror() {
        println!("res: {}", e.to_string());
        println!("cause: {}", e.source().unwrap().to_string());
    }
}
```

`impl Error for MyError`只需要实现`Debug`，`std::fmt::Display`即可。Error提供了`source`函数，能够通过这个函数将一些报错跨越抽象层传递上来。

## 5 Copy 和 Clone 的区别

官方手册在这里：<https://doc.rust-lang.org/std/clone/trait.Clone.html>

* Copy是为了告诉编译器，在`a = b`时使用copy语义而不是move语义。
* Copy是自动进行的，而Clone需要程序员主动使用。
* Copy是对栈上的数据进行按位复制，一个结构体，只有当它全部的成员均支持Copy时，该结构体才支持Copy。Clone是对堆上的数据进程复制，由用户自己保证数据复制的有效性。
* Copy、Clone之间不是简单的深拷贝、浅拷贝的关系，实际上两个均应该被看作是深拷贝。
* 实现Copy必须要实现Clone，即#[derive(Clone, Copy)]。原因是rust的规定：当实现了Copy时，如果调用a.clone()，这时clone的含义必须是简单内存拷贝，不能由用户自己随便实现Clone。Types that are Copy should have a trivial implementation of Clone. More formally: if `T: Copy`, `x: T`, and `y: &T`, then `let x = y.clone();` is equivalent to `let x = *y;`.

整体来讲：Copy使用的要求更加苛刻，但能用的肯定不会出问题，用起来简单。Clone要求没有那么苛刻，一切责任，用户自负。

一个clone的神奇的用法：去掉&。

对一个`a: &T`类型的数据，`a.clone()`的类型可以转换为`T`，源码：

```rs
pub trait Clone: Sized {
    // Required method
    fn clone(&self) -> Self;

    // Provided method
    fn clone_from(&mut self, source: &Self) { ... }
}
```

## last: 代码风格

### Result、Option的值转换

Rust可以快速使用map、map_or等一大类的语法糖进行快速的`Result<T, E>`、`Option<V>`的值转换。但是这些东西在我看来很容易混淆，专门记录一下：

* or: 重点在于会提供一个缺省值给Err()
* or_else：重点在于提供一个匿名函数给Err()调用

#### map_or_else

* 输入：提供两个匿名函数，同时转换T、E。
* 输出：看匿名函数，可以不输出返回值。

#### map_err

* 输入：提供一个匿名函数，转换E。
* 输出：Result

#### map_or

* 输入：提供一个匿名函数，转换T。提供一个缺省值应对E。
* 输出：匿名函数和缺省值必须是相同的类型。

#### or

* 输入：提供一个缺省值应对E。
* 输出：将E转换为对应的缺省值。

#### or_else

* 输入：提供一个匿名函数，转换E为T或E。
* 输出：Result
* 注意：和map_err类似，区别是：map_err的匿名函数只能将E转换为E，而or_else可以将E转换为T。

#### unwrap_or_else

* 输入：提供一个匿名函数，转换E。
* 输出：对于Ok，直接调用unwrap()；否则使用匿名函数做转换。

#### ok_or_else（只能对Option转换）

* 记忆：Option<V> => Result<T, E>
* 输入：提供一个匿名函数，转换None。
* 输出：将Some转换成Ok返回；None使用匿名函数转换为E。

### 返回值的处理

```rust
// 如果关注返回值,后面会用到:
let fd = match socket::socket(self.sock_addr.family(), self.sa_type, falgs, self.protocol) {
    Ok(fd) => fd,
    Err(why) => {
        println!("......");
        return Err(why);
    }
};

// 如果不关注返回值:
if let Err(why) = socket::setsockopt(fd, ReuseAddr, &true) {
    println!("......");
    return Err(why);
}
```
