---
title: Rust入坑指南：步步为营
date: 2020-02-21 00:05:43
tags: Rust
---

俗话说：“测试写得好，奖金少不了。”<!-- more -->

有经验的开发人员通常会通过单元测试来保证代码基本逻辑的正确性。如果你是一名新手开发者，并且还没体会到单元测试的好处，那么建议你先读一下我之前的一篇文章[代码洁癖系列（七）：单元测试的地位](https://jackeyzhe.github.io/2018/09/04/%E4%BB%A3%E7%A0%81%E6%B4%81%E7%99%96%E7%B3%BB%E5%88%97%EF%BC%88%E4%B8%83%EF%BC%89%EF%BC%9A%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95%E7%9A%84%E5%9C%B0%E4%BD%8D/)。

写单元测试一般需要三个步骤：

1. 准备测试用例，测试用例要能覆盖尽可能多的代码
2. 执行需要测试的代码
3. 判断结果，是否是你希望得到的结果

了解了这些以后，我们就来看看在Rust中应该怎么写单元测试。

首先我们建立一个library项目

``` bash
$ cargo new adder --lib
     Created library `adder` project
```

然后在src/lib.rs文件中开始写测试代码

``` rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```

此时在命令行运行` cargo test`就会得到测试结果

![测试结果1](https://res.cloudinary.com/dxydgihag/image/upload/v1582124140/Blog/rust/09/rust9-1.png)

可以看到，结果显示，Rust运行了一项测试并且测试通过。后面的Doc-tests我们先放下，以后再聊。

当然，这并不是我们常见的测试，在日常开发中，我们通常是先写我们的业务代码然后再对各个函数进行单元测试，最后还会对某个模块进行集成测试。那么我们就来模拟一下日常开发过程中应该如何来写测试。

### 单元测试

我们仍然是用上面的项目，先来在src/lib.rs中写一段“业务代码”

``` rust
pub fn add_two(a: i32) -> i32 {
    internal_adder(a, 2)
}

fn internal_adder(a: i32, b: i32) -> i32 {
    a + b
}
```

这是一段非常简单的代码，对外暴露的函数只是一个加2的功能，内部调用了一个两数相加的函数。现在我们就对这个内部函数做一个单元测试。

``` rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn internal() {
        assert_eq!(4, internal_adder(2, 2));
    }
}
```

在测试模块中，如果想要使用我们业务代码中的函数，就需要通过` use super::*;`将其引入可用范围。接着，还是执行` cargo test`，测试结果与刚才类似。

![测试结果2](https://res.cloudinary.com/dxydgihag/image/upload/v1582125779/Blog/rust/09/rust9-2.png)

测了半天全是通过的没什么意思，单元测试真正的作用是要发现代码中的问题，所以我们来尝试一个错误的试一下。假设我们希望2+2等于5。

![测试结果3](https://res.cloudinary.com/dxydgihag/image/upload/v1582127769/Blog/rust/09/rust9-3.png)

这里我们的assert_eq!左右不相等，引起了线程恐慌，因此导致测试失败。结果中给出了失败的原因，引起失败的位置，并且有一句提示：` note: run with RUST_BACKTRACE=1 environment variable to display a backtrace.` 我们按照这个提示，设置变量RUST_BACKTRACE=1，此时再执行`cargo test`。

![错误栈](https://res.cloudinary.com/dxydgihag/image/upload/v1582128445/Blog/rust/09/rust9-4.png)

Rust就会将错误栈打印出来，根据结果提示，这并不是完整的错误栈，我们还可以将RUST_BACKTRACE设置为full来查看更加详细的信息。这里我就不做演示了。

### 集成测试

接下来我们再演示一下集成测试。我们通常将集成测试单独放到一个目录中，在lib.rs文件中，rust识别测试mod的名称是tests，同样的，我们在src下创建tests目录。**tests**目录下就是我们的所有集成测试代码。

![测试目录](https://res.cloudinary.com/dxydgihag/image/upload/v1582129708/Blog/rust/09/rust9-5.png)

如图，integration_test是我们测试代码的文件，common目录下的mod.rs文件中是一些集成测试必要的配置。这里我们只是放了一个空的setup函数。

在集成测试中，我们就要像正常他人使用我们的代码时那样来进行测试，首先需要将我们的mod引入到可用范围，当然还需要加上common的mod。

``` rust
use adder;

mod common;

#[tests]
fn it_adds_two() {
    common::setup();
    assert_eq!(4, adder::add_two(2));
}
```

接着就可以测试我们对外暴露的函数了。

![集成测试](https://res.cloudinary.com/dxydgihag/image/upload/v1582130013/Blog/rust/09/rust9-6.png)

ok，集成测试的方法我们也掌握了。现在来看看一直被我们忽略的Doc-tests吧。

### 文档测试

我们已经知道，Rust中的注释是双斜线`//`，像我们刚刚写的library代码，如果想要把它发布到crate.io上让别人使用，那么我们就需要增加相应的文档，这里文档的每行都应该是三斜线`///`开头，而文档中也应该放一些例子供他人参考。（注意：下面注释中的代码需要包含在markdown的代码块格式中，这里写上三个`的话文档格式会乱掉。。。运行测试代码时请自行补充）

``` rust
/// Adds two to the number given.
///
/// # Examples
///
/// 
/// let arg = 5;
/// let answer = adder::add_two(arg);
///
/// assert_eq!(7, answer);
/// 
pub fn add_two(a: i32) -> i32 {
    internal_adder(a, 2)
}
```

现在我给add_two函数加上了文档，我们再次执行`cargo test`命令。

![文档测试](https://res.cloudinary.com/dxydgihag/image/upload/v1582131276/Blog/rust/09/rust9-7.png)

现在我们就明白了，Doc-tests测试就是运行我们文档中的例子。

### 常用特性

到目前为止，我们已经知道了在Rust中如何写测试代码了。接下来我们再来了解几个比较常用的特性。

#### 运行指定的测试代码

我们在开发过程中肯定不会每次都去跑全量的单元测试，那样太浪费时间了。通常是我们开发完一个功能之后，编写对应的单元测试，然后单独跑这个测试。那么Rust中能不能单独跑一个单元测试呢？答案是肯定的。

相信细心的同学已经发现了，Rust测试结果中，是针对每个测试单独统计结果，并且每个测试都有自己的名字，像我们前面写的`it_works`和`internal`。假设我们的代码中同时存在这两个函数，如果你想要单独跑internal这一个测试，就可以使用`cargo test internal`命令。

你也可以使用这种方法来执行多个名称类似的测试，假如我们有名称为`internal_a`的测试，那么执行`cargo test internal`命令时它也会被执行。

#### 忽略某个测试

当我们有一个测试执行时间非常长的时候，我们一般不会轻易去执行，这时如果你想要执行多个测试，除了用我们上面提到的方法，去指定不同的名称列表以外。还可以把这个测试忽略掉。

现在我不想执行`internal`测试了，只需要对代码进行如下改动：

``` rust
#[test]
#[ignore]
fn internal() {
  assert_eq!(4, internal_adder(2, 2));
}
```

这时再来运行测试，结果如图所示。

![忽略测试](https://res.cloudinary.com/dxydgihag/image/upload/v1582212742/Blog/rust/09/rust9-8.png)

我们发现此时`internal`测试已经被忽略了。

#### 测试异常情况

除了测试代码逻辑正常的情况，我们有时还需要测试一些异常情况，比如接收到非法参数时程序能否返回我们希望看到的异常。

我们首先来看一下如何测试程序返回异常信息。

Rust为我们提供了一个叫做should_panic的注解。我们可以使用它来测试程序是否返回异常：

``` rust
pub fn add_two(a: i32) -> i32 {
    internal_adder(a, 2)
}

fn internal_adder(a: i32, b: i32) -> i32 {
    if a < 0 {
        panic!("a should bigger than 0");
    }
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn internal() {
        assert_eq!(4, internal_adder(-2, 2));
    }
}
```

此时我们运行测试时就会发现internal测试通过，因为它发生了线程恐慌，这是我们希望看到的结果。

![测试异常](https://res.cloudinary.com/dxydgihag/image/upload/v1582213602/Blog/rust/09/rust9-9.png)

另外，我们还可以再指定我们具体期望的异常，那么就可以在should_panic后面加上expected参数。

``` rust
#[test]
#[should_panic(expected = "a should be positive")]
fn internal() {
  assert_eq!(4, internal_adder(-2, 2));
}
```

大家可以自行运行一下这段测试代码看看效果。

### 总结

文中我向大家介绍了在Rust中如何进行单元测试、集成测试，还有比较特殊的文档测试。最后还介绍了3种常见的测试特性。

最后想友情提醒大家一下，在开发过程中，不要写完一堆功能后再开始写单元测试，这时你很有可能会因为测试代码过于繁琐而放弃。建议大家每写一个功能，随即开始进行单元测试，这样也能立即看到自己的代码的执行效果，提高成就感。这就是所谓的“步步为营”。