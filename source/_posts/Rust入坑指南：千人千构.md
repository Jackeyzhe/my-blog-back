---
title: Rust入坑指南：千人千构
date: 2019-10-27 21:17:07
tags: Rust
---

坑越来越深了，在坑里的同学让我看到你们的双手！<!-- more -->

前面我们聊过了Rust最基本的几种数据类型。不知道你还记不记得，如果不记得可以先复习一下。上一个坑挖好以后，有同学私信我说坑太深了，下来的时候差点崴了脚。我只能对他说抱歉，下次还有可能更深。不过这篇文章不会那么深了，本文我将带大家探索Structs和Enums这两个坑，没错，是双坑。是不是很惊喜？好了，言归正传。我们先来介绍Structs。

### Structs

Structs在许多语言里都有，是一种自定义的类型，可以类比到Java中的类。Rust中使用Structs使用的是struct关键字。例如我们定义一个用户类型。

``` rust
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}
```

初始化时可以直接将上面对应的数据类型替换为正确的值。

``` rust
fn build_user(email: String, username: String) -> User {
    User {
        email: email,
        username: username,
        active: true,
        sign_in_count: 1,
    }
}
```

下面仔细观察这`email: email`和`username: username`这两行代码，有没有觉得有点麻烦？，如果User的所有属性值都是从函数参数传进来，那么我们每个参数名都要重复一遍。还好Rust为我们提供了语法糖，可以省去一些代码。

#### 初始化Struct时省去变量名

对于上面的初始化代码，我们可以做一些简化。

``` rust
fn build_user(email: String, username: String) -> User {
    User {
        email,
        username,
        active: true,
        sign_in_count: 1,
    }
}
```

你可以认为这是Rust的一个语法糖，当变量名和字段名相同时，初始化Struct的时候就可以省略变量名。让开发者不必做过多无意义的重复工作（写两遍email）。

#### 在其他实例的基础上创建Struct

除了上面的语法糖以外，在创建Struct时，Rust还提供了另一个语法糖，例如我们新建一个user2，它只有邮箱和用户名与user1不同， 其他属性都相同，那么我们可以使用如下代码：

``` rust
#![allow(unused_variables)]
fn main() {
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

let user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    active: true,
    sign_in_count: 1,
};

let user2 = User {
    email: String::from("another@example.com"),
    username: String::from("anotherusername567"),
    ..user1
};
}
```

这里的`..user1`表示剩下的字段的值都和user1相同。

#### Tuple Struct

接下来再来介绍两个特殊形式的Struct，一种是Tuple Struct，定义时与Tuple相似

``` rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);
```

它与Tuple的不同在于，你可以赋予Tuple Struct一个有意义的名字，而不只是无意义的一堆值。需要注意的是，这里我们定义的Color和Point是两种不同的类型，它们之间不能互相赋值。另外，如果你想要取得Tuple Struct中某个字段的值，和Tuple一样，使用`.`即可。

#### 空字段Struct

这里还有一种特殊的Struct，即没有字段的Struct。它叫做类单元结构（unit-like structs）。这种结构体一般用于实现某些特征，但又没有需要存储的数据。

#### Struct 方法

方法和函数非常相似，不同之处在于，定义方法时，必须有与之关联的Struct，并且方法的第一个参数必须是self。我们先来看一下如何定义一个方法：

``` rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}
```

我们提到，方法必须与Struct关联，这里使用`impl`关键字定义一段指定Struct的实现代码，然后在这个代码块中定义Struct相关的方法，注意我们的area方法符合规则，第一个参数是self。调用时只需要用`.`就可以。

``` rust
fn main() {
    let rect1 = Rectangle { width: 30, height: 50 };
	rect1.area();
}
```

这里的`&self`其实是代替了` rectangle: &Rectangle `，至于这里为什么要使用&符号，我们在[前文]( [https://jackeyzhe.github.io/2019/10/13/Rust%E5%85%A5%E5%9D%91%E6%8C%87%E5%8D%97%EF%BC%9A%E6%A0%B8%E5%BF%83%E6%A6%82%E5%BF%B5/](https://jackeyzhe.github.io/2019/10/13/Rust入坑指南：核心概念/) )已经做了介绍。当然，这里self也不是必须要加&符号，你可以认为它是一个正常的参数，根据需要来使用。

有些同学可能会有些困惑，我们已经有了函数了，为什么还要使用方法？这其实主要是为了代码的结构。我们需要将Struct实例可以做的操作都放到impl实现代码块中，方便修改和查找。而使用函数则可能存在开发人员随便找个位置来定义的尴尬情况，这对于后期维护代码的开发人员来讲将是一种灾难。

现在我们已经知道，方法必须定义在impl代码块中，且第一个参数必须是self，但有时你会在Impl代码块中看到第一个参数不是self的，而且Rust也允许这种行为。

``` rust
impl Rectangle {
    fn square(size: u32) -> Rectangle {
        Rectangle { width: size, height: size }
    }
}
```

这是什么情况？刚才说的不对？其实不然，这种函数叫做相关函数（associated functions）。它仍然是函数，而不是方法并且直接和Struct相关，类似于Java中的静态方法。调用时直接使用双冒号（`::`），我们之前见过很多次的`String::from("Hi")`就是String的相关函数。

最后提一点，Rust支持为一个Struct定义多个实现代码块。但是我们并不推荐这样使用。

至此，第一个坑Struct就挖好了，接下来就是第二个坑Enum。

### Enum

很多编程语言都支持枚举类型，Rust也不例外。因此枚举对于大部分开发人员来说并不陌生，这里我们简单介绍一些使用方法及特性。

先来看一下Rust中如何定义枚举和获取枚举值。

``` rust
enum IpAddrKind {
    V4,
    V6,
}

let six = IpAddrKind::V6;
let four = IpAddrKind::V4;
```

这里的例子只是最简单的定义枚举的方法，每个枚举的值也可以关联其他类型的的值。例如

``` rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

此外，Enum也可以像Struct拥有impl代码块，你也可以在里面定义方法。

#### Option枚举

Option是Rust标准库中定义的一个枚举。如果你用过Java8的话，一定知道一个Optional类，专门用来处理null值。Rust中是不存在null值的，因为它太容易引起bug了。但如果确实需要的时候怎么办呢，这就需要Option枚举登场了。我们先来看一看它的定义：

``` rust
enum Option<T> {
    Some(T),
    None,
}
```

很简单对不对。它是一个枚举，只有两个值，一个是Some，一个是None，其中Some还关联了一个类型T的值，这个T类似于Java中的泛型，即它可以是任意类型。

在使用时，可以直接使用Some或None，前面不用加`Option::`。当你使用None时，必须要指定T的具体类型。

``` rust
let some_number = Some(5);
let some_string = Some("a string");

let absent_number: Option<i32> = None;
```

需要注意的是Option\<T\>与T并不是相同的类型。你可以在[官方文档]( https://doc.rust-lang.org/std/option/enum.Option.html )中查看从Option\<T\>中提取出T的方法。

#### match流程控制

Rust有一个很强大的流程控制操作叫做match，它有些类似于Java中的switch。首先匹配一系列的模式，然后执行相应的代码。与Java中switch不同的是，switch只能支持数值/枚举类型（现在也可以支持字符串），match可以支持任意类型。

``` rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

此外，match还可以支持模式中绑定值。

``` rust
enum UsState {
    Alabama,
    Alaska,
    // --snip--
}

enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(UsState),
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("State quarter from {:?}!", state);
            25
        },
    }
}
```

#### match与Option\<T\>

前面我们聊到了从Option\<T\>中提取T的值，我们来介绍一种通过match提取的方法。

``` rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}

let five = Some(5);
let six = plus_one(five);
let none = plus_one(None);
```

这种方法在参数中必须声明T的具体类型，这里再思考一个问题，如果我们确定x一定不会是None，那么可不可以去掉None的那个条件？

#### `_`占位符

答案是不可以，Rust要求match必须列举出所有可能的条件。例如，如果一个u8类型的，就需要列举0到255这些条件。这样做的话，可能一天也写不了几个match语句吧。所以Rust又给我们准备了一个语法糖。

针对上述情况，就可以写成下面这样：

``` rust
let some_u8_value = 0u8;
match some_u8_value {
    1 => println!("one"),
    3 => println!("three"),
    5 => println!("five"),
    7 => println!("seven"),
    _ => (),
}
```

我们只需要列举我们关心的几种情况，然后用占位符`_`表示剩余所有情况。看到这我只想感叹一句，这糖真甜啊。

#### if let

对于我们只关心一个条件的match来讲，还有一种更加简洁的语法，那就是if let。

举个栗子，我们只想要Option\<u8\>中值为3时打印相关信息，利用我们已经掌握的知识，可以这样写。

``` rust
let some_u8_value = Some(0u8);
match some_u8_value {
    Some(3) => println!("three"),
    _ => (),
}
```

如果用if let呢，就会更加简洁一些。

``` rust
if let Some(3) = some_u8_value {
    println!("three");
}
```

这里要注意，当match只有一个条件时，才可以使用if let替代。

有同学可能会问，既然叫if let，那么有没有else条件呢？答案是有的。对于下面这种情况

``` rust
let mut count = 0;
match coin {
    Coin::Quarter(state) => println!("State quarter from {:?}!", state),
    _ => count += 1,
}
```

如果替换成if let语句，应该是

``` rust
let mut count = 0;
if let Coin::Quarter(state) = coin {
    println!("State quarter from {:?}!", state);
} else {
    count += 1;
}
```

### 总结

第二个坑也挖好了，来总结一下吧。本文我们首先介绍了Struct，它类似于Java中的类，可以供开发人员自定义类型。然后介绍了两种初始化Struct时的简化代码的方法。接着是定义Struct相关的方法。在介绍完Struct以后，紧接着又介绍了大家都很熟悉的Enum枚举类型。重点说了Rust中特殊的枚举Option，然后介绍了match和if let这两种流程控制语法。

最后，按照国际惯例，我还是要诚挚的邀请你早日入坑。坑里真的是冬暖夏凉~