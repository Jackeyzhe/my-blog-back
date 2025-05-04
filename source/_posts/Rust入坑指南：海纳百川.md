---
title: Rust入坑指南：海纳百川
date: 2020-01-14 00:24:45
tags: Rust
---

今天来聊Rust中两个重要的概念：泛型和trait。很多编程语言都支持泛型，Rust也不例外，相信大家对泛型也都比较熟悉，它可以表示任意一种数据类型。trait同样不是Rust所特有的特性，它借鉴于Haskell中的Typeclass。简单来讲，Rust中的trait就是对类型行为的抽象，你可以把它理解为Java中的接口。<!-- more -->

### 泛型

在前面的文章中，我们其实已经提及了一些泛型类型。例如Option<T>、Vec<T>和Result<T, E>。泛型可以在函数、数据结构、Enum和方法中进行定义。在Rust中，我们习惯使用T作为通用的类型名称，当然也可以是其他名称，只不过习惯上优先使用T（Type）来表示。它可以帮我们消除一些重复代码，例如实现逻辑相同但参数类型不同的两个函数，我们就可以通过泛型技术将其进行合并。下面我们分别演示泛型的几种定义。

#### 在函数中定义

泛型在函数的定义中，可以是参数，也可以是返回值。前提是必须要在函数名的后面加上<T>。

``` rust
fn largest<T>(list: &[T]) -> T {
```

#### 在数据结构中定义

如果数据结构中某个字段可以接收任意数据类型，那么我们可以把这个字段的类型定义为T，同样的，为了让编译器认识这个T，我们需要在结构体名称后边标识一下。

``` rust
struct Point<T> {
    x: T,
    y: T,
}
```

上面的例子中，x和y都是可以接受任意类型，但是，它们两个的类型必须相同，如果传入的类型不同，编译器仍然会报错。那如果想要让x和y能够接受不同的类型应该怎么办呢？其实也很简单，我们定义两种不同的泛型就好了。

``` rust
struct Point<T, U> {
    x: T,
    y: U,
}
```

#### 在Enum中定义

在Enum中定义泛型我们已经接触过比较多了，最常见的例子就是Option<T>和Result<T, E>。其定义方法也和在数据结构中的定义方法类似

``` rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

#### 在方法中定义

我们在实现定义了泛型的数据结构或Enum时，方法中也可以定义泛型。例如我们对刚刚定义的Point<T>进行实现。

``` rust
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
```

可以看到，我们的方法返回值的类型是T的引用，为了让编译器识别T，我们必须要在```impl```后面加上```<T>```。

另外，我们在对结构体进行实现时，也可以实现指定的类型，这样就不需要在```impl```后面加标识了。

``` rust
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

了解了泛型的几种定义之后，你有没有想过一个问题：Rust中使用泛型会对程序运行时的性能造成不良影响吗？答案是不会，因为Rust对于泛型的处理都是在编译阶段进行的，对于我们定义的泛型，Rust编译器会对其进行单一化处理，也就是说，我们定义一个具有泛型的函数（或者其他什么的），Rust会根据需要将其编译为具有具体类型的函数。

``` rust
let integer = Some(5);
let float = Some(5.0);
```

例如我们的代码使用了这两种类型的Option，那么Rust编译器就会在编译阶段生成两个指定具体类型的Option。

``` rust
enum Option_i32 {
    Some(i32),
    None,
}

enum Option_f64 {
    Some(f64),
    None,
}
```

这样我们在运行阶段直接使用对应的Option就可以了，而不需要再进行额外复杂的操作。所以，如果我们泛型定义并使用的范围很大，也不会对运行时性能造成影响，受影响的只有编译后程序包的大小。

### Trait

Trait可以说是Rust的灵魂，Rust中所有的抽象都是依靠Trait来实现的。

我们先来看看如何定义一个Trait。

``` rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

定义trait使用了关键字```trait```，后面跟着trait的名称。其内容是trait的「行为」，也就是一个个函数。但是这里的函数没有实现，而是直接以```;```结尾。不过这这并不是必须的，Rust也支持下面这种写法：

``` rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```

对于这样的写法，它表示summarize函数的默认实现。

#### Trait的实现

上面是一种默认实现，接下来我们介绍一下在Rust中，对一个Trait的常规实现。Trait的实现是需要针对结构体的，即我们要写明是哪个结构体的哪种行为。

``` rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

上述代码中，我们分别定义了结构体NewArticle和Tweet，然后为它们实现了trait，定义了summarize函数对应的逻辑。

#### 作为参数的Trait

此外，trait还可以作为函数的参数，也就是需要传入一个实现了对应trait的结构体的实例。

``` rust
pub fn notify(item: impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

作参数时，我们需要使用```impl```关键字来定义参数类型。

Rust还提供了另一种语法糖来，即**Trait限定**，我们可以使用泛型约束的语法来限定Trait参数。

``` rust
pub fn notify<T: Summary>(item: T) {
    println!("Breaking news! {}", item.summarize());
}
```

如上述代码，我们可以通过Trait来限定泛型T的范围。这样的语法糖可以在多个参数的函数中帮助我们简化代码。下面两行代码就有比较明显的对比

``` rust
pub fn notify(item1: impl Summary, item2: impl Summary) {

pub fn notify<T: Summary>(item1: T, item2: T) {
```

如果某个参数有多个trait限定，就可以使用```+```来表示

``` rust
pub fn notify<T: Summary + Display>(item: T) {
```

如果我们有更多的参数，并且有每个参数都有多个trait限定，及时我们使用了上面这种语法糖，代码仍然有些繁杂，会降低可读性。所以Rust又为我们提供了```where```关键字。

``` rust
fn some_function<T, U>(t: T, u: U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{
```

它帮助我们在函数定义的最后写一个trait限定列表，这样可以使代码的可读性更高。

#### Trait作为返回值

``` rust
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: String::from("horse_ebooks"),
        content: String::from("of course, as you probably already know, people"),
        reply: false,
        retweet: false,
    }
}
```

Trait作为返回值类型，和作为参数类似，只需要在定义返回类型时使用``` impl Trait```。

### 总结

本文我们简单介绍了泛型和Trait，包括它们的定义和使用方法。泛型主要是针对数据类型的一种抽象，而Trait则是对数据类型行为的一种抽象，Rust中并没有严格意义上的继承，多是用组合的形式。这也体现了「多组合，少继承」的设计思想。

最后留个预告，这个坑还没完，我们下次继续往深处挖。