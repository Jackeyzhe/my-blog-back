---
title: Rust入坑指南：齐头并进（下）
date: 2020-03-23 21:24:02
tags: Rust
---

前文中我们聊了Rust如何管理线程以及如何利用Rust中的锁进行编程。今天我们继续学习并发编程。<!-- more -->

### 原子类型

许多编程语言都会提供原子类型，Rust也不例外，在前文中我们聊了Rust中锁的使用，有了锁，就要小心死锁的问题，Rust虽然声称是安全并发，但是仍然无法帮助我们解决死锁的问题。原子类型就是编程语言为我们提供的无锁并发编程的最佳手段。熟悉Java的同学应该知道，Java的编译器并不能保证代码的执行顺序，编译器会对我们的代码的执行顺序进行优化，这一操作成为指令重排。而Rust的多线程内存模型不会进行指令重排，它可以保证指令的执行顺序。

通常来讲原子类型会提供以下操作：

- Load：从原子类型读取值
- Store：为一个原子类型写入值
- CAS（Compare-And-Swap）：比较并交换
- Swap：交换
- Fetch-add（sub/and/or）：表示一系列的原子的加减或逻辑运算

Ok，这些基础的概念聊完以后，我们就来看看Rust为我们提供了哪些原子类型。Rust的原子类型定义在标准库`std::sync::atomic`中，目前它提供了12种原子类型。

![原子类型](https://res.cloudinary.com/dxydgihag/image/upload/v1584877213/Blog/rust/13/rust13-1.png)

下面这段代码是Rust演示了如何用原子类型实现一个自旋锁。

``` rust
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::thread;

fn main() {
    let spinlock = Arc::new(AtomicUsize::new(1));
    let spinlock_clone = spinlock.clone();
    let thread = thread::spawn(move|| {
        spinlock_clone.store(0, Ordering::SeqCst);
    });
    while spinlock.load(Ordering::SeqCst) != 0 {}
    if let Err(panic) = thread.join() {
        println!("Thread had an error: {:?}", panic);
    }
}
```

我们利用AtomicUsize的store方法将它的值设置为0，然后用load方法获取到它的值，如果不是0，则程序一直空转。在store和load方法中，我们都用到了一个参数：`Ordering::SeqCst`，在声明中能看出来它也是属于atomic包。

我们在文档中发现它是一个枚举。其定义为

``` rust
pub enum Ordering {
    Relaxed,
    Release,
    Acquire,
    AcqRel,
    SeqCst,
}
```

它的作用是将内存顺序的控制权交给开发者，我们可以自己定义底层的内存排序。下面我们一起来看一下这5种排序分别代表什么意思

- Relaxed：表示「没有顺序」，也就是开发者不会干预线程顺序，线程只进行原子操作
- Release：对于使用Release的store操作，在它之前所有使用Acquire的load操作都是可见的
- Acquire：对于使用Acquire的load操作，在它之前的所有使用Release的store操作也都是可见的
- AcqRel：它代表读时使用Acquire顺序的load操作，写时使用Release顺序的store操作
- SeqCst：使用了SeqCst的原子操作都必须先存储，再加载。

一般情况下建议使用SeqCst，而不推荐使用Relaxed。

### 线程间通信

Go语言文档中有这样一句话：**不要使用共享内存来通信，应该使用通信实现共享内存。**

Rust标准库选择了CSP并发模型，也就是依赖channel来进行线程间的通信。它的定义是在标准库`std::sync::mpsc`中，里面定义了三种类型的CSP进程：

- Sender：发送异步消息
- SyncSender：发送同步消息
- Receiver：用于接收消息

我们通过一个栗子来看一下channel是如何创建并收发消息的。

``` rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

首先，我们先是使用了`channel()`函数来创建一个channel，它会返回一个（Sender, Receiver）元组。它的缓冲区是无界的。此外，我们还可以使用`sync_channel()`来创建channel，它返回的则是（SyncSender, Receiver）元组，这样的channel发送消息是同步的，并且可以设置缓冲区大小。

接着，在子线程中，我们定义了一个字符串变量，并使用`send()`函数向channel中发送消息。这里send返回的是一个Result类型，所以使用unwrap来传播错误。

在main函数最后，我们又用`recv()`函数来接收消息。

这里需要注意的是，`send()`函数会转移所有权，所以，如果你在发送消息之后再使用val变量时，程序就会报错。

现在我们已经掌握了使用Channel进行线程间通信的方法了，这里还有一段代码，感兴趣的同学可以自己执行一下这段代码看是否能够顺利执行。如果不能，应该怎么修改这段代码呢？

``` rust
use std::thread;
use std::sync::mpsc;
fn main() {
    let (tx, rx) = mpsc::channel();
    for i in 0..5 {
        let tx = tx.clone();
        thread::spawn(move || {
            tx.send(i).unwrap();
        });
    }

    for rx in rx.iter() {
        println!("{:?}", j);
    }
}
```

### 线程池

在实际工作中，如果每次都要创建新的线程，每次创建、销毁线程的开销就会变得非常可观，甚至会成为系统性能的瓶颈。对于这种问题，我们通常使用线程池来解决。

Rust的标准库中没有现成的线程池给我们使用，不过还是有一些第三方库来支持的。这里我使用的是[threadpool](https://crates.io/crates/threadpool)。

首先需要在Cargo.toml中增加依赖`threadpool = "1.7.1"`。然后就可以使用`use threadpool::ThreadPool;`将ThreadPool引入我们的程序中了。

``` rust
use threadpool::ThreadPool;
use std::sync::mpsc::channel;

fn main() {
    let n_workers = 4;
    let n_jobs = 8;
    let pool = ThreadPool::new(n_workers);

    let (tx, rx) = channel();
    for _ in 0..n_jobs {
        let tx = tx.clone();
        pool.execute(move|| {
            tx.send(1).expect("channel will be there waiting for the pool");
        });
    }

    assert_eq!(rx.iter().take(n_jobs).fold(0, |a, b| a + b), 8);
}
```

这里我们使用`ThreadPool::new()`来创建一个线程池，初始化4个工作线程。使用时用`execute()`方法就可以拿出一个线程来进行具体的工作。

### 总结

今天我们介绍了Rust并发编程的三种特性：原子类型、线程间通信和线程池的使用。

原子类型是我们进行无锁并发的重要手段，线程间通信和线程池也都是工作中所必须使用的。当然并发编程的知识远不止于此，大家有兴趣的可以自行学习也可以与我交流讨论。