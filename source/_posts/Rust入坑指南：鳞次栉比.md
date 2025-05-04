---
title: Rust入坑指南：鳞次栉比
date: 2019-11-27 23:28:33
tags: Rust
---

很久没有挖Rust的坑啦，今天来挖一些排列整齐的坑。没错，就是要介绍一些集合类型的数据类型。“鳞次栉比”这个标题是不是显得很有文化？<!-- more -->

在[Rust入坑指南：常规套路]( https://jackeyzhe.github.io/2019/10/08/Rust入坑指南：常规套路/ )一文中我们已经介绍了一些基本数据类型了，它们都存储在栈中，今天我们重点介绍3种数据类型：string，vector和hash map。

### String

String类型我们在之前的学习中已经有了较多的接触，但是没有进行过详细的介绍。有些有编程基础的同学可能不屑于学习String类型，毕竟它在所有编程语言中可以说是最常用的类型了，大家也都很熟悉了。对于有这种心理的同学，我想对他们说：我刚开始也是这样想的，直到后来我被编译器揍的满头包，才下定决心回来认真学习一下String类型。

Rust的字符串分为以下几种类型：

- **str**：表示固定长度的字符串
- **String**：表示可增长的字符串
- **CStr**：表示由C分配，被Rust借用的字符串，一般用于和C语言交互
- **CString**：表示由Rust分配并且可以传递给C语言的字符串
- **OsStr**：表示和操作系统相关的字符串，主要为了兼容Windows
- **OsString**：OsStr的可变版本
- **Path**：表示路径
- **PathBuf**：是Path的可变版本

本文我们重点讨论前两种，因为它们是开发过程中最常用的，也是比较容易混淆的。对于str，我们常见的是它的引用类型，``` &str```。如果你看过了[Rust入坑指南：核心概念]( [https://jackeyzhe.github.io/2019/10/13/Rust%E5%85%A5%E5%9D%91%E6%8C%87%E5%8D%97%EF%BC%9A%E6%A0%B8%E5%BF%83%E6%A6%82%E5%BF%B5/](https://jackeyzhe.github.io/2019/10/13/Rust入坑指南：核心概念/) )一文后，相信你已经了解了引用类型和Ownership的概念。也就是说String类型具有Ownership而&str没有。

在Rust中，String本质上是Vec\<u8>，Vec是向量集合的关键字，我们在后面会介绍。String类型由三个部分组成，分别是：指向堆中字节序列的指针，记录堆中字节序列的长度和堆分配的容量。通过一段代码也许你很有更深的理解。

``` rust
fn main() {
    let mut a = String::from("foo");
    println!("{:p}", a.as_ptr());
    println!("{:p}", &a);
    assert_eq!(a.len(), 3);
    a.reserve(10);
    assert_eq!(a.capacity(), 13);
}
```

在这段代码中我们可以看到，a.as_ptr()获取指针和&a获取的指针是不一样的。

![rust06-1](https://res.cloudinary.com/dxydgihag/image/upload/v1575123441/Blog/rust/06/rust06-1.png)

这里我们解释一下，as_ptr获取到的指针是堆中字节序列的指针地址，而&a的地址是字符串变量在栈上的指针地址。另外，len()和capacity()方法得到的长度都是字节数量，而非字符数量。这里你可以自己动手试试中文字符的长度。

聊完了字符串的基本概念以后，相信你已经对Rust的字符串有了一个大概的认识。接下来我们就一起来看一看字符串的CRUD的方法吧。

#### 创建字符串

话不多说，先来展示一下创建字符串的各种方法。

``` rust
fn main() {
    let string: String = String::new();
    let string: String = String::from("hello rust");
    let string: String = String::with_capacity(10);
    let str: &'static str = "Jackey";
    let string: String = str.to_owned();
    let string: String = str.to_string();
}
```

我们比较常用的是前两种，下面介绍一下后面几个方法。with_capacity()是创建一个空字符串，参数表示在堆中分配的字节数。to_owned和to_string是演示了如何把&str类型转换成String类型。

#### 修改字符串

Rust修改字符串的常用方法也有很多，例如在字符串后追加，连接两个字符串，更新字符串等。下面这段代码就展示了一些修改字符串的方法。

``` rust
fn main() {
    let mut hello = String::from("Hello, ");
    hello.push('J');    // 追加单个字符
    hello.push_str("ackey! ");    //追加字符串
    println!("push: {}", hello);

    hello.extend(['M', 'y', ' '].iter());   //追加多个字符，参数为迭代器
    hello.extend("name".chars());
    println!("extend: {}", hello);

    hello.insert(0, 'h');   //类似于push，可以指定插入的位置
    hello.insert(1, 'a');
    hello.insert(2, '!');
    hello.insert_str(0, "Haha");
    println!("insert: {}", hello);

    let left = "Hello, ".to_string();
    let right = "World".to_string();
    let result = left + &right;
    println!("+: {}", result);   //使用+连接字符串时，第二个必须为引用
    let mut message = "rust".to_string();   //使用+=连接字符串时，字符串必须定义为可变
    message += "!";
    println!("+=: {}", message);

    let s = String::from("foobar");
    let s: String = s
        .chars()
        .enumerate()
        .map(|(_i, c)| {c.to_uppercase().to_string()})
        .collect();
    println!("update chars: {}", s);
  
    let s1 = String::from("hello");
    let s2 = String::from("rust");
    let s3 = format!("{}-{}", s1, s2);
    println!("format: {}", s3);
}
```

我们对上面的代码做一些补充的解释。

push和insert类似，带有_str的方法接收的参数是字符串，否则只能接收单个字符。insert可以指定插入的位置，而push只能在字符串末尾插入。

使用「+」连接字符串时，第一个参数是String类型，第二个则需要是引用类型&str。这类似于我们调用一个add方法，它的定义是这样的：

``` rust
fn add(self, s: &str) -> String {
```

所以，第一个参数的ownership转移到了函数中，又通过返回结果传递出来。也就是说，在使用了+操作符之后，``` left```已经没有ownership了。

#### 字符串查找

在Rust中，字符串是不能根据位置来获取到指定字符的。也就是下面这段代码是编译不过的。

``` rust
let s1 = String::from("hello");
let h = s1[0];
```

因为，Rust会认为这个0是指第一个字节，而Rust字符串中的字符可能占有多个字节（还记得前面我让你用中文字符实验代码吗？）所以，如果你单纯的想要获取一个字节，编译器不知道你是真的想要获取字节对应的数值，还是要获取那个字符。

我们在处理字符串时通常有以下方法：

``` rust
fn main() {
    let hello = "Здравствуйте";
    let s = &hello[0..4];
    println!("{}", s);

    let chars = hello.chars();
    for c in chars {
        println!("{}", c);
    }

    let bytes = hello.bytes();
    for byte in bytes {
        println!("{}", byte);
    }

    let get = hello.get(0..1);
    let mut s = String::from("hello");
    let get_mut = s.get_mut(3..5);

    let message = String::from("hello-world");
    let (left, right) = message.split_at(6);
    println!("left: {}, right: {}", left, right);
}
```

通常是使用字符切片，也可以使用chars方法获取到Chars迭代器，然后可以对每个字符进行单独处理。此外，使用get或get_mut方法也可以接收索引范围，返回指定的字符串切片。返回结果是Option类型，这是因为如果指定的索引返回不能返回完整字符，那么Rust就会返回None。这里也可以使用is_char_boundary方法来判断一个位置是否是非法边界。

最后，也可以使用split_at或split_at_mut方法来分割字符串。这要求分割的位置正好是字符边界位置，如果不是，程序就会崩溃。

#### 删除字符串

Rust的标准库提供了一些删除字符串的方法，我们来演示一些：

``` rust
fn main() {
    let mut hello = String::from("hello");
    hello.remove(3);
    println!("remove: {}", hello);
    hello.pop();
    println!("pop: {}", hello);
    hello.truncate(1);
    println!("truncate: {}", hello);
    hello.clear();
    println!("clear: {}", hello);
}
```

结果如图：

![rust06-2](https://res.cloudinary.com/dxydgihag/image/upload/v1575211017/Blog/rust/06/rust06-2.png)

remove方法用来删除字符串中的某个字符，其接收的参数是字符的起始位置，如果是不是某个字符的起始位置，会导致程序崩溃。

pop方法会弹出字符串末尾的字符，truncate方法是截取指定长度字符串，而clear方法则是用来清空字符串。

至此，关于Rust中的字符串的基本概念和CRUD我们都已经介绍完了，接下来我们再来看另一种集合类型Vector。

### Vector

Vector是用来存储相同数据类型的多个数据一种数据类型。它的关键字是``` Vec<T> ```。下面我们一起来看看向量的CRUD吧。

#### 创建向量

``` rust
fn main() {
    let v1: Vec<i32> = Vec::new();
    let v2 = vec![1, 2, 3];
}
```

上面这段代码演示了创建一个向量的两种方式，第一种是使用new函数来创建一个空的向量，由于没有添加元素，所以要显式的指定存储元素的类型。第二种是创建一个有初始值的向量集合，我们直接使用vec！宏，然后指定初始值即可，不需要指定向量中元素的数据类型，因为编译器可以自己推断出来。

#### 更新向量

``` rust
fn main() {
    let mut v = Vec::new();
    v.push(1);
    v.push(2);
}
```

创建一个空的向量之后，如果我们想要增加元素，就可以直接使用push方法，向末尾追加元素。

#### 删除向量

``` rust
fn main() {
    let mut v = Vec::new();
    v.push(1);
    v.push(2);
    v.push(3);

    v.pop();
    for i in &v {
        println!("{}", i);
    }
}
```

删除单个元素可以使用pop方法，而要删除整个向量，只能像其他结构体一样，到其ownership失效。

#### 读取向量元素

``` rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];

    let third: &i32 = &v[2];
    println!("The third element is {}", third);

    match v.get(2) {
        Some(third) => println!("The third element is {}", third),
        None => println!("There is no third element."),
    }

    let v = vec![100, 32, 57];
    for i in &v {
        println!("{}", i);
    }
}
```

当你需要读取单个指定元素时，有两种方法可以用，一种是使用```[]```，另一种是使用get方法。两种方法的区别是：第一种返回的是元素的类型，而get返回的是Option类型。如果你指定的位置越界了，那么使用第一种方法程序会直接崩溃，而使用第二种方法则会返回None。

此外，还可以通过遍历向量的形式来读取元素。如果想要存储不同类型的数据，我们可以借助枚举类型。

``` rust
fn main() {
    enum SpreadsheetCell {
        Int(i32),
        Float(f64),
        Text(String),
    }

    let row = vec![
        SpreadsheetCell::Int(3),
        SpreadsheetCell::Text(String::from("blue")),
        SpreadsheetCell::Float(10.12),
    ];
}
```

### HashMap

HashMap存储了KV结构的数据，各个Key必须是同一种类型，各个Value必须是同一种类型。由于HashMap是三种集合类型中使用最少的，所以在使用之前，需要手动引入进来

``` rust
use std::collections::HashMap;
```

#### 创建HashMap

首先我们来了解一下如何创建一个新的Hash Map并增加元素。

``` rust
use std::collections::HashMap;
fn main() {
    let field_name = String::from("Favorite color");
    let field_value = String::from("Blue");

    let mut map = HashMap::new();
    map.insert(field_name, field_value);
}
```

注意，在使用insert方法时，```field_name```和```field_value```都会失去所有权。那如何再使用它们呢？我们只能从Hash Map中再拿出来。

#### 访问Hash Map的数据

``` rust
use std::collections::HashMap;
fn main() {
    let field_name = String::from("Favorite color");
    let field_value = String::from("Blue");

    let mut map = HashMap::new();
    map.insert(field_name, field_value);

    let favorite = String::from("Favorite color");
    let color = map.get(&favorite);
    match color {
        Some(x) => println!("{}", x),
        None => println!("None"),
    }
}
```

可以看到，我们使用get可以获取到指定Key的值，get方法返回的是Option类型，如果没有指定的Value，则会返回None。此外，也可以使用for循环来遍历Hash Map。

``` rust
use std::collections::HashMap;
fn main() {
    let mut scores = HashMap::new();

    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);

    for (key, value) in &scores {
        println!("{}: {}", key, value);
    }
}
```

#### 更新Hash Map

当我们向同一个Key insert值时，旧的值就会被覆盖。如果只想要在Key不存在时插入，则可以使用entry。

``` rust
use std::collections::HashMap;
fn main() {
    let mut scores = HashMap::new();
    scores.insert(String::from("Blue"), 10);

    scores.entry(String::from("Yellow")).or_insert(50);
    scores.entry(String::from("Blue")).or_insert(50);

    println!("{:?}", scores);
}
```

### 总结

今天带大家一起挖了三个坑，string，vector和hash map，分别介绍了每种数据类型的CRUD。对string的介绍占了比较大的篇幅，因为它是最常用的数据类型之一。当然这部分的相关知识还有很多，欢迎大家和我一起学习交流。