---
title: Rust入坑指南：齐头并进（上）
date: 2020-03-15 22:30:49
tags: Rust
---

我们知道，如今CPU的计算能力已经非常强大，其速度比内存要高出许多个数量级。为了充分利用CPU资源，多数编程语言都提供了并发编程的能力，Rust也不例外。<!-- more -->

聊到并发，就离不开多进程和多线程这两个概念。其中，进程是资源分配的最小单位，而线程是程序运行的最小单位。线程必须依托于进程，多个线程之间是共享进程的内存空间的。进程间的切换复杂，CPU利用率低等缺点让我们在做并发编程时更加倾向于使用多线程的方式。

当然，多线程也有缺点。其一是程序运行顺序不能确定，因为这是由内核来控制的，其二就是多线程编程对开发者要求比较高，如果不充分了解多线程机制的话，写出的程序就非常容易出Bug。

多线程编程的主要难点在于如何保证线程安全。什么是线程安全呢？因为多个线程之间是共享内存空间的，因此就会存在同时对相同的内存进行写操作，那就会出现写入数据互相覆盖的问题。如果多个线程对内存只有读操作，没有任何写操作，那么也就不会存在安全问题，我们可以称之为线程安全。

常见的并发安全问题有**竞态条件**和**数据竞争**两种，竞态条件是指多个线程对相同的内存区域（我们称之为临界区）进行了“读取-修改-写入”这样的操作。而数据竞争则是指一个线程写一个变量，而另一个线程需要读这个变量，此时两者就是数据竞争的关系。这么说可能不太容易理解，不过不要紧，待会儿我会举两个具体的例子帮助大家理解。不过在此之前，我想先介绍一下Rust中是如何进行并发编程的。

### 管理线程

在Rust标准库中，提供了两个包来进行多线程编程：

- std::thread，定义一些管理线程的函数和一些底层同步原语
- std::sync，定义了锁、Channel、条件变量和屏障

我们使用std::thread中的`spawn`函数来创建线程，它的使用非常简单，其参数是一个闭包，传入创建的线程需要执行的程序。

``` rust
use std::thread;
use std::time::Duration;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```

这段代码中，我们有两个线程，一个主线程，一个是用`spawn`创建出来的线程，两个线程都执行了一个循环。循环中打印了一句话，然后让线程休眠1毫秒。它的执行结果是这样的：

![执行结果](https://res.cloudinary.com/dxydgihag/image/upload/v1584253897/Blog/rust/12/rust12-1.png)

从结果中我们能看出两件事：第一，两个线程是交替执行的，但是并没有严格的顺序，第二，当主线程结束时，它并没有等子线程运行完。

那我们有没有办法让主线程等子线程执行结束呢？答案当然是有的。Rust中提供了`join`函数来解决这个问题。

``` rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
        thread::sleep(Duration::from_millis(1));
    }

    handle.join().unwrap();
}
```

这样主线程就必须要等待子线程执行完毕。

在某些情况下，我们需要将一些变量在线程间进行传递，正常来讲，闭包需要捕获变量的引用，这里就涉及到了生命周期问题，而子线程的闭包的存活周期有可能长于当前的函数，这样就会造成悬垂指针，这在Rust中是绝对不允许的。因此我们需要使用`move`关键字将所有权转移到闭包中。

``` rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("Here's a vector: {:?}", v);
    });

    handle.join().unwrap();
}
```

使用`thread::spawn`创建线程是不是非常简单。但是也是因为它的简单，所以可能无法满足我们一些定制化的需求。例如制定线程的栈大小，线程名称等。这时我们可以使用`thread::Builder`来创建线程。

``` rust
use std::thread::{Builder, current};

fn main() {
    let mut v = vec![];
    for id in 0..5 {
        let thread_name = format!("child-{}", id);
        let size: usize = 3 * 1024;
        let builder = Builder::new().name(thread_name).stack_size(size);
        let child = builder.spawn(move || {
            println!("in child:{}", current().name().unwrap());
        }).unwrap();
        v.push(child);
    }

    for child in v {
        child.join().unwrap_or_default();
    }
}
```

我们使用`thread::spawn`创建的线程返回的类型是`JoinHandle<T>`，而使用`builder.spawn`返回的是`Result<JoinHandle<T>>`，因此这里需要加上`unwrap`方法。

除了刚才提到了这些函数和结构体，`std::thread`还提供了一些底层同步原语，包括park、unpark和yield_now函数。其中park提供了阻塞线程的能力，unpark用来恢复被阻塞的线程。yield_now函数则可以让线程放弃时间片，让给其他线程执行。

### Send和Sync

聊完了线程管理，我们再回到线程安全的话题，Rust提供的这些线程管理工具看起来和其他没有什么区别，那Rust又是如何保证线程安全的呢？

秘密就在`Send`和`Sync`这两个trait中。它们的作用是：

- Send：实现Send的类型可以安全的在线程间传递所有权。
- Sync：实现Sync的类型可以安全的在线程间传递不可变借用。

现在我们可以看一下`spawn`函数的源码

``` rust
#[stable(feature = "rust1", since = "1.0.0")]
pub fn spawn<F, T>(f: F) -> JoinHandle<T> where
    F: FnOnce() -> T, F: Send + 'static, T: Send + 'static
{
    Builder::new().spawn(f).expect("failed to spawn thread")
}
```

其参数F和返回值类型T都加上了`Send + 'static`限定，Send表示闭包必须实现Send，这样才可以在线程间传递。而`'static`表示T只能是非引用类型，因为使用引用类型则无法保证生命周期。

在[Rust入坑指南：智能指针](https://jackeyzhe.github.io/2020/03/09/Rust%E5%85%A5%E5%9D%91%E6%8C%87%E5%8D%97%EF%BC%9A%E6%99%BA%E8%83%BD%E6%8C%87%E9%92%88/)一文中，我们介绍了共享所有权的指针`Rc<T>`，但在多线程之间共享变量时，就不能使用`Rc<T>`，因为它的内部不是原子操作。不过不要紧，Rust为我们提供了线程安全版本：`Arc<T>`。

下面我们一起来验证一下。

``` rust
use std::thread;
use std::rc::Rc;

fn main() {
    let mut s = Rc::new("Hello".to_string());
    for _ in 0..3 {
        let mut s_clone = s.clone();
        thread::spawn(move || {
            s_clone.push_str(" world!");
        });
    }
}
```

这个程序会报如下错误

![Rc报错](https://res.cloudinary.com/dxydgihag/image/upload/v1584278809/Blog/rust/12/rust12-2.png)

那我们把`Rc`替换为`Arc`试一下。

``` rust
use std::sync::Arc;
...
let mut s = Arc::new("Hello".to_string());
```

很遗憾，程序还是报错。

![Arc报错](https://res.cloudinary.com/dxydgihag/image/upload/v1584279037/Blog/rust/12/rust12-3.png)

这是因为，Arc默认是不可变的，我们还需要提供内部可变性。这时你可能想到来RefCell，但是它也是线程不安全的。所以这里我们需要使用`Mutex<T>`类型。它是Rust实现的互斥锁。

### 互斥锁

Rust中使用`Mutex<T>`实现互斥锁，从而保证线程安全。如果类型T实现了Send，那么`Mutex<T>`会自动实现Send和Sync。它的使用方法也比较简单，在使用之前需要通过`lock`或`try_lock`方法来获取锁，然后再进行操作。那么现在我们就可以对前面的代码进行修复了。

``` rust
use std::thread;
use std::sync::{Arc, Mutex};

fn main() {
    let mut s = Arc::new(Mutex::new("Hello".to_string()));
    let mut v = vec![];
    for _ in 0..3 {
        let s_clone = s.clone();
        let child = thread::spawn(move || {
            let mut s_clone = s_clone.lock().unwrap();
            s_clone.push_str(" world!");
        });
        v.push(child);
    }

    for child in v {
        child.join().unwrap();
    }
}
```

### 读写锁

介绍完了互斥锁之后，我们再来了解一下Rust中提供的另外一种锁——读写锁`RwLock<T>`。互斥锁用来独占线程，而读写锁则可以支持多个读线程和一个写线程。

在使用读写锁时要注意，读锁和写锁是不能同时存在的，在使用时必须要使用显式作用域把读锁和写锁隔离开。

### 总结

本文我们先是介绍了Rust管理线程的两个函数：`spawn`、`join`。并且知道了可以使用Builder结构体定制化创建线程。然后又学习了Rust提供线程安全的两个trait，Send和Sync。最后我们一起学习了Rust提供的两种锁的实现：互斥锁和读写锁。

关于Rust并发编程坑还没有到底，接下来还有条件变量、原子类型这些坑等着我们来挖。今天就暂时歇业了。