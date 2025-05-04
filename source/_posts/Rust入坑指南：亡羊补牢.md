---
title: Rust入坑指南：亡羊补牢
date: 2019-12-30 22:39:27
tags: Rust
---

如果你已经开始学习Rust，相信你已经体会过Rust编译器的强大。它可以帮助你避免程序中的大部分错误，但是编译器也不是万能的，如果程序写的不恰当，还是会发生错误，让程序崩溃。所以今天我们就来聊一聊Rust中如何处理程序错误，也就是所谓的“亡羊补牢”。<!-- more -->

### 基础概念

在编程中遇到的非正常情况通常可以分为三类：失败、错误、异常。

Rust中用两种方式来消除失败：强大的类型系统和断言。

对于类型系统，熟悉Java的同学应该比较清楚。例如我们给一个接收参数为int的函数传入了字符串类型的变量。这是由编译器帮我们处理的。

![rust07-1](https://res.cloudinary.com/dxydgihag/image/upload/v1577629475/Blog/rust/07/Rust07-1.png)

关于断言，Rust支持6种断言。分别是：

- assert!
- assert_eq!
- assert_ne!
- debug_assert!
- debug_assert_eq!
- debug_assert_ne!

从名称我们就可以看出来这6种断言，可以分为两大类，带debug的和不带debug的，它们的区别就是assert开头的在调试模式和发布模式下都可以使用，而debug开头的只可以在调试模式下使用。再来解释每个大类下的三种断言，assert!是用于断言布尔表达式是否为true，assert_eq!用于断言两个表达式是否相等，assert_ne!用于断言两个表达式是否不相等。当不符合条件时，断言会引发线程恐慌（panic!）。

Rust处理异常的方法有4种：Option<T>、Result<T, E>、线程恐慌（Panic）、程序终止（Abort）。接下来我们对这些方法进行详细介绍。

### Option<T>

Option<T>我们在[Rust入坑指南：千人千构](https://jackeyzhe.github.io/2019/10/27/Rust%E5%85%A5%E5%9D%91%E6%8C%87%E5%8D%97%EF%BC%9A%E5%8D%83%E4%BA%BA%E5%8D%83%E6%9E%84/)一文中我们进行过一些介绍，它是一种枚举类型，主要包括两种值：Some(T)和None，Rust也是靠它来避免空指针异常的。

在前文中，我们并没有详细介绍如何从Option<T>中提取出T，其实最基本的，可以用match来提取。而我也在前文中给你提供了官方文档的链接，不知道你有没有看。如果还没来得及看也没有关系，我把我看到的一些方法分享给你。

这里介绍两种方法，一种是expect，另一种是unwrap系列的方法。我们通过一个例子来感受一下。

``` rust
fn main() {
    let a = Some("a");
    let b: Option<&str> = None;
    assert_eq!(a.expect("a is none"), "a");
    assert_eq!(b.expect("b is none"), "b is none");  //匹配到None会引起线程恐慌，打印的错误是expect的参数信息

    assert_eq!(a.unwrap(), "a");   //如果a是None，则会引起线程恐慌
    assert_eq!(b.unwrap_or("b"), "b"); //匹配到None时返回指定值
    let k = 10;
    assert_eq!(Some(4).unwrap_or_else(|| 2 * k), 4);// 与unwrap_or类似，只不过参数是FnOnce() -> T
    assert_eq!(None.unwrap_or_else(|| 2 * k), 20);
}
```

这是从Option<T>中提取值的方法，有时我们会觉得每次处理Option<T>都需要先提取，然后再做相应计算这样的操作比较麻烦，那么有没有更加高效的操作呢？答案是肯定的，我从文档中找到了map和and_then这两种方法。

其中map方法和unwrap一样，也是一系列方法，包括map、map_or和map_or_else。map会执行参数中闭包的规则，然后将结果再封为Option<T>并返回。

``` rust
fn main() {
    let some_str = Some("Hello!");
    let some_str_len = some_str.map(|s| s.len());
    assert_eq!(some_str_len, Some(6));
}
```

但是，如果参数本身返回的结果就是Option的话，处理起来就比较麻烦，因为每执行一次map都会多封装一层，最后的结果有可能是Some(Some(Some(...)))这样N多层Some的嵌套。这时，我们就可以用and_then来处理了。

利用and_then方法，我们就可以有如下的链式调用：

``` rust
fn main() {
    assert_eq!(Some(2).and_then(sq).and_then(sq), Some(16));
}

fn sq(x: u32) -> Option<u32> { 
    Some(x * x) 
}
```

关于Option<T>我们就先聊到这里，大家只需要记住，它可以用来处理空值，然后能够使用它的一些处理方法就可以了，实在记不住这些方法，也可以在用的时候再去[文档](https://doc.rust-lang.org/std/option/enum.Option.html)中查询。

### Result<T, E>

聊完了Option<T>，我们再来看另一种错误处理方法，它也是一个枚举类型，叫做Result<T, E>，定义如下：

``` rust
#[must_use = "this `Result` may be an `Err` variant, which should be handled"]
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

实际上，Option<T>可以被看作Result<T, ()>。从定义中我们可以看到Result<T, E>有两个变体：Ok(T)和Err(E)。

Result<T, E>用于处理真正意义上的错误，例如，当我们想要打开一个不存在的文件时，或者我们想要将一个非数字的字符串转换为数字时，都会得到一个Err(E)结果。

Result<T, E>的处理方法和Option<T>类似，都可以使用unwrap和expect方法，也可以使用map和and_then方法，并且用法也都类似，这里就不再赘述了。具体的方法使用细节可以自行查看[官方文档](https://doc.rust-lang.org/std/result/enum.Result.html)。

这里我们来看一下如何处理不同类型的错误。

Rust在std::io模块定义了统一的错误类型Error，因此我们在处理时可以分别匹配不同的错误类型。

``` rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {:?}", e),
            },
            ErrorKind::PermissionDenied => panic!("Permission Denied!"),
            other_error => panic!("Problem opening the file: {:?}", other_error),
        },
    };
}
```

在处理Result<T, E>时，我们还有一种处理方法，就是**try!**宏。它会使代码变得非常精简，但是在发生错误时，会将错误返回，传播到外部调用函数中，所以我们在使用之前要考虑清楚是否需要传播错误。

对于上面的代码，使用try!宏就会非常精简。

``` rust
use std::fs::File;

fn main() {
    let f = try!(File::open("hello.txt"));
}
```

try!使用起来虽然简单，但也有一定的问题。像我们刚才提到的传播错误，再就是有可能出现多层嵌套的情况。因此Rust引入了另一个语法糖来代替try!。它就是问号操作符“**?**”。

``` rust
use std::fs::File;
use std::io;
use std::io::Read;

fn main() {
    read_username_from_file();
}

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

问号操作符必须在处理错误的代码后面，这样的代码看起来更加优雅。

### 恐慌（Panic）

我们从最开始就聊到线程恐慌，那道理什么是恐慌呢？

在Rust中，无法处理的错误就会造成线程恐慌，手动执行**panic!**宏时也会造成恐慌。当程序执行panic!宏时，会打印相应的错误信息，同时清理堆栈并退出。但是栈回退和清理会花费大量的时间，如果你想要立即终止程序，可以在Cargo.toml文件中```[profile]```区域中增加```panic = 'abort' ```，这样当发生恐慌时，程序会直接退出而不清理堆栈，内存空间都由操作系统来进行回收。

程序报错时，如果你想要查看完整的错误栈信息，可以通过设置环境变量``` RUST_BACKTRACE=1```的方式来实现。

如果程序发生恐慌，我们前面所说的Result<T, E>就不能使用了，Rust为我们提供了catch_unwind方法来捕获恐慌。

``` rust
use std::panic;

fn main() {
    let result = panic::catch_unwind(|| {panic!("crash and burn")});
    assert!(result.is_err());
    println!("{}", 1 + 2);
}
```

在上面这段代码中，我们手动执行一个panic宏，正常情况下，程序会在第一行退出，并不会执行后面的代码。而这里我们用了catch_unwind方法对panic进行了捕获，结果如图所示。

![rust07-2](https://res.cloudinary.com/dxydgihag/image/upload/v1577715961/Blog/rust/07/rust07-2.png)

Rust虽然打印了恐慌信息，但是并没有影响程序的执行，我们的代码``` println!("{}", 1 + 2);```可以正常执行。

### 总结

至此，Rust处理错误的方法我们已经基本介绍完了，为什么说是基本介绍完了呢？因为还有一些大佬开发了一些第三方库来帮助我们更加方便的处理错误，其中比较有名的有error-chain和failure，这里就不做过多介绍了。

通过本节的学习，相信你的Rust程序一定会变得更加健壮。