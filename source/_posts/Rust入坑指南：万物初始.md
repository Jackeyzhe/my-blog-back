---
title: Rust入坑指南：万物初始
date: 2020-04-08 23:02:34
tags: Rust
---

有没有同学记得我们一起挖了多少个坑？嗯…其实我自己也不记得了，今天我们再来挖一个特殊的坑，这个坑可以说是挖到根源了——**元编程**。<!-- more -->

元编程是编程领域的一个重要概念，它允许程序将代码作为数据，在运行时对代码进行修改或替换。如果你熟悉Java，此时是不是想到了Java的反射机制？没错，它就是属于元编程的一种。

### 反射

Rust也同样支持反射，Rust的反射是由标准库中的`std::any::Any`包支持的。

这个包中提供了以下几个方法

![Any包的方法](https://res.cloudinary.com/dxydgihag/image/upload/v1586171714/Blog/rust/15/rust15-1.png)

TypeId是Rust中的一种类型，它被用来表示某个类型的唯一标识。`type_id(&self)`这个方法返回变量的TypeId。

`is()`方法则用来判断某个函数的类型。

可以看一下它的源码实现

``` rust
pub fn is<T: Any>(&self) -> bool {
  let t = TypeId::of::<T>();

  let concrete = self.type_id();

  t == concrete
}
```

可以看到它的实现非常简单，就是对比TypeId。

`downcast_ref()`和`downcast_mut()`是一对用于将泛型T转换为具体类型的方法。其返回的类型是`Option<&T>`和`Option<&mut T>`，也就是说`downcast_ref()`将类型T转换为不可变引用，而`downcast_mut()`将T转换为可变引用。

最后我们通过一个例子来看一下这几个函数的具体使用方法。

``` rust
use std::any::{Any, TypeId};

fn main() {
    let v1 = "Jackey";
    let mut a: &Any;
    a = &v1;
    println!("{:?}", a.type_id());
    assert!(a.is::<&str>());


    print_any(&v1);
    let v2: u32 = 33;
    print_any(&v2);
}

fn print_any(any: &Any) {
    if let Some(v) = any.downcast_ref::<u32>() {
        println!("u32 {:x}", v);
    } else if let Some(v) = any.downcast_ref::<&str>() {
        println!("str {:?}", v);
    } else {
        println!("else");
    }
}
```

### 宏

Rust的反射机制提供的功能比较有限，但是Rust还提供了宏来支持元编程。

到目前为止，宏对我们来说是一个既熟悉又陌生的概念，熟悉是因为我们一直在使用`println!`宏，陌生则是因为我们从没有详细介绍过它。

对于`println!`宏，我们直观上的使用感受是它和函数差不多。但两者之间还是有一定的区别的。

我们知道对于函数，它接收参数的个数是固定的，并且在函数定义时就已经固定了。而宏接收的参数个数则是不固定的。

这里我们说的宏都是类似函数的宏，此外，Rust还有一种宏是类似于属性的宏。它有点类似于Java中的注解，通常作为一种标记写在函数名上方。

``` rust
#[route(GET, "/")]
fn index() {
```

route在这里是用来指定接口方法的，对于这个服务来讲，根路径的`GET`请求都被路由到这个index函数上。这样的宏是通过属于**过程宏**，它的定义使用了`#[proc_macro_attribute]`注解。而函数类似的过程宏在定义时使用的注解是`#[proc_macro]`。

除了过程宏以外，宏的另一大分类叫做**声明宏**。声明宏是通过`macro_rules!`来声明定义的宏，它比过程宏的应用要更加广泛。我们曾经接触过的`vec!`就是声明宏的一种。它的定义如下：

``` rust
#[macro_export]
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

下面我们来定义一个属于自己的宏。

自定义宏需要使用`derive`注解。（例子来自the book）

我们先来创建一个叫做hello_macro的lib库，只定义一个trait。

``` rust
pub trait HelloMacro {
    fn hello_macro();
}
```

接着再创建一个子目录hello_macro_derive，在hello_macro_derive/Cargo.toml文件中添加依赖

``` rust
[lib]
proc-macro = true

[dependencies]
syn = "0.14.4"
quote = "0.6.3"
```

然后就可以在hello_macro_derive/lib.rs文件中定义我们自定义宏的功能实现了。

``` rust
extern crate proc_macro;

use crate::proc_macro::TokenStream;
use quote::quote;
use syn;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // Construct a representation of Rust code as a syntax tree
    // that we can manipulate
    let ast = syn::parse(input).unwrap();

    // Build the trait implementation
    impl_hello_macro(&ast)
}

fn impl_hello_macro(ast: &syn::DeriveInput) -> TokenStream {
    let name = &ast.ident;
    let gen = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}", stringify!(#name));
            }
        }
    };
    gen.into()
}
```

这里使用了两个crate：syn和quote，其中syn是把Rust代码转换成一种特殊的可操作的数据结构，而quote的作用则与它刚好相反。

可以看到，我们自定义宏使用的注解是`#[proc_macro_derive(HelloMacro)]`，其中HelloMacro是宏的名称，在使用时，我们只需要使用注解`#[derive(HelloMacro)]`即可。

在使用时我们应该先引入这两个依赖

``` rust
hello_macro = { path = "../hello_macro" }
hello_macro_derive = { path = "../hello_macro/hello_macro_derive" }
```

然后再来使用

``` rust
use hello_macro::HelloMacro;
use hello_macro_derive::HelloMacro;

#[derive(HelloMacro)]
struct Pancakes;

fn main() {
    Pancakes::hello_macro();
}
```

运行结果显示，我们能够成功在实现中捕获到结构体的名字。

![result](https://res.cloudinary.com/dxydgihag/image/upload/v1586190192/Blog/rust/15/rust15-2.png)

### 总结

我们在本文中先后介绍了Rust的两种元编程：反射和宏。其中反射提供的功能能力较弱，但是宏提供的功能非常强大。我们所介绍的宏的相关知识其实只是皮毛，要想真正理解宏，还需要花更多的时间学习。

