---
title: Rust入坑指南：有条不紊
date: 2019-11-03 21:51:04
tags: Rust
---

随着我们的坑越来越多，越来越大，我们必须要对各种坑进行管理了。Rust为我们提供了一套坑务管理系统，方便大家有条不紊的寻找、管理、填埋自己的各种坑。<!-- more -->

Rust提供给我们一些管理代码的特性：

- **Packages：**Cargo的一个特性，帮助你进行构建、测试和共享crates
- **Crates：**生成库或可执行文件的模块树
- **Modules**和**use：**用于控制代码组织、范围和隐私路径
- **Paths：**struct、function和module的命名方法

下面我们来具体看一下这些特性是如何帮助我们组织代码的。

### Packages和Crates

package可以理解为一个项目，而crate可以理解为一个代码库。crate可以供多个项目使用。那我们的项目中package和crate是怎么定义的呢？

之前我们总是通过IDEA来新建项目，今天我们换个方法，在命令行中使用cargo命令来创建。

``` bash
$ cargo new hello-world
     Created binary (application) `hello-world` package
$ ls hello-world
Cargo.toml
src
$ ls hello-world/src
main.rs
```

可以看到，我们使用cargo创建项目后，只有两个文件，Cargo.toml和src目录下的main.rs。

Cargo.toml是管理项目依赖的文件，每个Cargo.toml定义一个package。main.rs文件的存在表示package中包含一个二进制crate，它是二进制crate的入口文件，crate的名称和package相同。如果src目录下存在lib.rs文件，说明package中包含一个和package名称相同的库crate。

一个package可以包含多个二进制crate，它们由src/lib目录下的文件定义。如果你的项目想引用他人的crate，可以在Cargo.toml文件中增加依赖。每个crate都有自己的命名空间，因此如果你引入了一个crate里面定义了一个名为hello的函数，你仍然可以在自己的crate中再定义一个名为hello的函数。

### Module

Module帮助我们在crate中组织代码，同时Module也是封装代码的重要工具。接下来还是通过一个栗子来详细了解Module。

前面我们说过，库crate定义在src/lib.rs文件中。这里首先创建一个包含了库crate的package:

``` bash
cargo new --lib restaurant
```

然后在src中定义一些module和函数。

``` rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}
```

可以看到我们使用关键字`mod`来定义Module，Module中可以继续定义Module或函数。这样我们就可以比较方便的把相关的函数放到一个Module中，并为Module命名，提高代码的可读性。另外Module中还可以定义struct和枚举。由于Module中可以嵌套定义子Module，最终我们定义出来的代码类似一个树形。

那么如何访问Module中的函数呢？这就要提到Path了。这部分比较好理解，Module树相当于系统文件目录，而Path则是目录的路径。

### Path

这里的路径和系统文件路径一样，都分为相对路径和绝对路径两种。其中绝对路径必须以`crate`开头，因为它代码整个Module树的根节点。路径之间使用的是双冒号来表示引用。

现在我来尝试在一个函数中调用add_to_waitlist函数：

![05-1](https://res.cloudinary.com/dxydgihag/image/upload/v1572276180/Blog/rust/05/rust5-1.png)

可以看到这里不管用绝对路径还是相对路径都报错了，错误信息是模块hosting和函数add_to_waitlist是私有（private）的。我们先暂时放下这个错误，根据这里的错误提示，我们知道了当我们定义一个module时，默认情况下是私有的，我们可以通过这种方法来封装一些代码的实现细节。

OK，回到刚才的问题，那我们怎么才能解决这个错误呢？地球人都知道应该把对应的模块与函数公开出来。Rust中标识模块或函数为公有的关键字是`pub`。

我们用pub关键字来把对应的模块和函数公开

![05-2](https://res.cloudinary.com/dxydgihag/image/upload/v1572277111/Blog/rust/05/rust5-2.png)

这样我们就可以在module外来调用module内的函数了。

#### Rust中的私有规则

现在我们再回过头来看Rust中的一些私有规则，如果你试验了上面的例子，也许会有一些发现。

Rust中私有规则适用于所有项（函数、方法、结构体、枚举、模块和常量），它们默认都是私有的。父模块中的项不能访问子模块中的私有项，而子模块中的项可以访问其祖辈（父模块及以上）中的项。

#### Struct和Enum的私有性

Struct和Enum的私有性略有不同，对于Struct来讲，我可以只将其中的某些字段设置为公有的，其他字段可以仍然保持私有。

``` rust
mod back_of_house {
    pub struct Breakfast {
        pub toast: String,
        seasonal_fruit: String,
    }

    impl Breakfast {
        pub fn summer(toast: &str) -> Breakfast {
            Breakfast {
                toast: String::from(toast),
                seasonal_fruit: String::from("peaches"),
            }
        }
    }
}

pub fn eat_at_restaurant() {
    // Order a breakfast in the summer with Rye toast
    let mut meal = back_of_house::Breakfast::summer("Rye");
    // Change our mind about what bread we'd like
    meal.toast = String::from("Wheat");
    println!("I'd like {} toast please", meal.toast);
}
```

而对于Enum，如果一个Enum是公有的，那么它的所有值都是公有的，因为私有的值没有意义。

#### 相对路径和绝对路径的选择

这种选择不存在正确与否，只有是否合适。因此这里我们只是举例说明一些合适的情况。

我们仍以上述代码为例，如果我们可以预见到以后需要把front_of_house模块和eat_at_restaurant函数移动到一个新的名为customer_experience的模块中，就应该使用相对路径，这样我们就对其进行调整。

类似的，如果我们需要把eat_at_restaurant函数移动到dining模块中，那么我们选择绝对路径的话就不需要做调整。

综上，我们需要对代码的优化方向有一些前瞻性，并以此来判断需要使用相对路径还是绝对路径。

相对路径除了以当前模块开头外，还可以以super开头。它表示的是父级模块，类似于文件系统中的两个点(`..`)。

### use关键字

绝对路径和相对路径可以帮助我们找到指定的函数，但用起来也非常的麻烦，每次都要写一大长串路径。还好Rust为我们提供了use关键字。在很多语言中都有import关键字，这里的use就有些类似于import。不过Rust会提供更加丰富的用法。

use最基本的用法就是引入一个路径。我们就可以更加方便的使用这个路径下的一些方法：

``` rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
```

这个路径可以是绝对路径，也可以是相对路径，但如果是相对路径，就必须要以self开头。上面的例子可以写成：

``` rust
use self::front_of_house::hosting;
```

这与我们前面讲的相对路径似乎有些矛盾，Rust官方说会在之后的版本处理这个问题。

use还可以更进一步，直接指向具体的函数或Struct或Enum。但习惯上我们使用函数时，use后面使用的是路径，这样可以在调用函数时知道它属于哪个模块；而在使用Struct/Enum时，则具体指向它们。当然，这只是官方建议的编程习惯，你也可以有自己的习惯，不过最好还是按照官方推荐或者是项目约定的规范比较好。

对于同一路径下的某些子模块，在引入时可以合并为一行，例如：

``` rust
use std::io;
use std::cmp::Ordering;
// 等价于
use std::{cmp::Ordering, io};
```

有时我们还会遇到引用不同包下相同名称Struct的情况，这时有两种解决办法，一是不指定到具体的Struct，在使用时加上不同的路径；二是使用`as`关键字，为Struct起一个别名。

方法一：

``` rust
use std::fmt;
use std::io;

fn function1() -> fmt::Result {
    // --snip--
}

fn function2() -> io::Result<()> {
    // --snip--
}
```

方法二：

``` rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
}

fn function2() -> IoResult<()> {
    // --snip--
}

```

如果要导入某个路径下的全部模块或函数，可以使用`*`来表示。当然我是非常不建议使用这种方法的，因为导入全部的话，如果出现名称冲突就会很难排查问题。

对于外部的依赖包，我们需要先在Cargo.toml文件中添加依赖，然后就可以在代码中使用use来引入依赖库中的路径。Rust提供了一些标准库，即std下的库。在使用这些标准库时是不需要添加依赖的。

有些同学看到这里可能要开始抱怨了，说好了介绍怎么拆分文件，到现在还是在一个文件里玩，这不是欺骗读者嘛。

别急，这就开始拆分。

### 开始拆分

我们拿刚才的一段代码为例

``` rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}

```

首先我们可以把front_of_house模块下的内容拆分出去，需要在src目录下新建一个front_of_house.rs文件，然后把front_of_house模块下的内容写到文件中。lib.rs文件中，只需要声明front_of_house模块即可，不需要具体的定义。声明模块时，将花括号即内容改为分号就可以了。

``` rust
mod front_of_house;
```

然后我们可以继续拆分front_of_house模块下的hosting模块和serving模块，这时需要新建一个名为front_of_house的文件件，在该文件夹下放置要拆分的模块的同名文件，把模块定义的内容写在文件中，front_of_house.rs文件同样只保留声明即可。

拆分后的文件目录如图

![rust05-3](https://res.cloudinary.com/dxydgihag/image/upload/v1572367091/Blog/rust/05/rust5-3.png)

本文主要讲了Rust中Package、Crate、Module、Path的概念和用法，有了这些基础，我们后面才有可能开发一些比较大的项目。

ps：本文的代码示例均来自[the book](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html)。