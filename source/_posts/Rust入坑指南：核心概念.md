---
title: Rust入坑指南：核心概念
date: 2019-10-13 01:02:27
tags: Rust
---

如果说前面的坑我们一直在用小铲子挖的话，那么今天的坑就是用挖掘机挖的。<!-- more -->

今天要介绍的是Rust的一个核心概念：Ownership。全文将分为什么是Ownership以及Ownership的传递类型两部分。

### 什么是Ownership

每种编程语言都有自己的一套内存管理的方法。有些需要显式的分配和回收内存（如C），有些语言则依赖于垃圾回收器来回收不使用的内存（如Java）。而Rust不属于以上任何一种，它有一套自己的内存管理规则，叫做Ownership。

在具体介绍Ownership之前，我想要先声明一点。[Rust入坑指南：常规套路](https://jackeyzhe.github.io/2019/10/08/Rust%E5%85%A5%E5%9D%91%E6%8C%87%E5%8D%97%EF%BC%9A%E5%B8%B8%E8%A7%84%E5%A5%97%E8%B7%AF/)一文中介绍的数据类型，其数据都是存储在栈中。而像String或一些自定义的复杂数据结构（我们以后会对它们进行详细介绍），其数据则存储在堆内存中。明确了这一点后，我们来看下Ownership的规则有哪些。

#### Ownership的规则

- 在Rust中，每一个值都有对应的变量，这个变量称为值的owner
- 一个值在某一时刻只能有一个owner
- 当owner超出作用域后，值会被销毁

这三条规则非常重要，记住他们会帮助你更好的理解本文。

#### 变量作用域

Ownership的规则中，有一条是owner超过范围后，值会被销毁。那么owner的范围又是如何定义的呢？在Rust中，花括号通常是变量范围作用域的标志。最常见的在一个函数中，变量s的范围从定义开始生效，直到函数结束，变量失效。

```rust
fn main() {                      // s is not valid here, it’s not yet declared
    let s = "hello";   // s is valid from this point forward

    // do stuff with s
}                      // this scope is now over, and s is no longer valid
```

这个这和其他大多数编程语言很像，对于大多数编程语言，都是从变量定义开始，为变量分配内存。而回收内存则是八仙过海各显神通。对于有依赖GC的语言来说，并不需要关心内存的回收。而有些语言则需要显式回收内存。显式回收就会存在一定的问题，比如忘记回收或者重复回收。为了对开发者更加友好，Rust使用自动回收内存的方法，即在变量超出作用域时，回收为该变量分配的内存。

### Ownership的移动

前面我们提到，花括号通常是变量作用域隔离的标志（即Ownership失效）。除了花括号以外，还有其他的一些情况会使Ownership发生变化，先来看两段代码。

```rust
let x = 5;
let y = x;
println!("x: {}", x);
```

```rust
let s1 = String::from("hello");
let s2 = s1;
println!("s1: {}", s1);
```

*作者注：双冒号是Rust中函数引用的标志，上面的意思是引用String中的from函数，这个函数通常用来构建一个字符串对象。*

这两段代码看起来唯一的区别就是变量的类型，第一段使用的是整数型，第二段使用的是字符串型。而执行结果却是第一段可以正常打印x的值，第二段却报错了。这是什么原因呢？

我们来分析一下代码。对于第一段代码，首先有个整数值5，赋给了变量x，然后把x的值copy了一份，又赋值给了y。最后我们成功打印x。看起来比较符合逻辑。实际上Rust也是这么操作的。

对于第二段代码我们想象中，也可以是这样的过程，但实际上Rust并不是这样做的。先来说原因：对于较大的对象来说，这样的复制是非常浪费空间和时间的。那么Rust中实际情况是怎么样呢？

首先，我们需要了解Rust中String类型的结构：

![String结构](https://res.cloudinary.com/dxydgihag/image/upload/v1570639634/Blog/rust/03/trpl04-01.svg)

上图中左侧是String对象的结构，包括指向内容的指针、长度和容量。这里长度和容量相同，我们暂时先不关注。后面详细介绍String类型时会提到两者的区别。这部分内容都存储在栈内存中。右侧部分是字符串的内容，这部分存储在堆内存中。

有的朋友可能想到了，既然复制内容会造成资源浪费，那我只复制结构这部分好了，内容再多，我复制的内容长度也是可控的，而且也是在栈中复制，和整数类型类似。这个方法听起啦不错，我们来分析一下。按照上面这种说法，内存结构大概是这个样子。

![String-2](https://res.cloudinary.com/dxydgihag/image/upload/v1570721759/Blog/rust/03/trpl04-02.svg)

这种会有什么问题呢？还记得Ownership的规则吗？owner超出作用域时，回收其数据所占用的内存。在这个例子中，当函数执行结束时，s1和s2同时超出作用域，那么上图中右侧这块内存就会被释放两次。这也会产生不可预知的bug。

Rust为了解决这一问题，在执行`let s2 = s1;`这句代码时，认为s1已经超出了作用域，即右侧的内容的owner已经变成了s2，也可以说s1的ownership转移给了s2。也就是下图所示的情况。

![String-real](https://res.cloudinary.com/dxydgihag/image/upload/v1570722292/Blog/rust/03/trpl04-03.svg)

#### 另一种实现：clone

如果你确实需要深度拷贝，即复制堆内存中的数据。Rust也可以做到，它提供了一个公共方法叫做clone。

```rust
let s1 = String::from("hello");
let s2 = s1.clone();

println!("s1 = {}, s2 = {}", s1, s2);
```

clone的方法执行后，内存结构如下图：

![String-clone](https://res.cloudinary.com/dxydgihag/image/upload/v1570722578/Blog/rust/03/trpl04-04.svg)

#### 函数间转移

前面我们聊到的是Ownership在String之间转移，在函数间也是一样的。

```rust
fn main() {
    let s = String::from("hello");  // s 作用域开始

    takes_ownership(s);             // s's 的值进入函数
                                    // ... s在这里已经无效

} // s在这之前已经失效
fn takes_ownership(some_string: String) { // some_string 作用域开始
    println!("{}", some_string);
} // some_string 超出作用域并调用了drop函数
  // 内存被释放
```

那有没有办法在执行takes_ownership函数后使s继续生效呢？一般我们会想到在函数中将ownership还回来。然后很自然的就想到我们之前介绍的函数的返回值。既然传参可以转移ownership，那么返回值应该也可以。于是我们可以这样操作：

```rust
fn main() {
    let s1 = String::from("hello");     // s2 comes into scope

    let s2 = takes_and_gives_back(s1);  // s1 被转移到函数中
                                        // takes_and_gives_back，
    									// 将ownership还给s2
} // s2超出作用域，内存被回收，s1在之前已经失效


// takes_and_gives_back 接收一个字符串然后返回一个
fn takes_and_gives_back(a_string: String) -> String { // a_string 开始作用域

    a_string  // a_string 被返回，ownership转移到函数外
}
```

这样做是可以实现我们的需求，但是有点太麻烦了，幸好Rust也觉得这样很麻烦。它为我们提供了另一种方法：引用（*references*）。

### 引用和借用

引用的方法很简单，只需要加一个`&`符。

```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

这种形式可以在没有ownership的情况下访问某个值。其原理如下图：

![references](https://res.cloudinary.com/dxydgihag/image/upload/v1570724291/Blog/rust/03/trpl04-05.svg)

这个例子和我们在前面写的例子很相似。仔细观察会发现一些端倪。主要有两点不同：

1. 在传入参数的时候，s1前面加了&符。这意味着我们创建了一个s1的引用，它并不是数据的owner，因此在它超出作用域时也不会销毁数据。
2. 函数在接收参数时，变量类型String前也加了&符。这表示参数要接收的是一个字符串的引用对象。

我们把函数中接收引用的参数称为借用。就像实际生活中我写完了作业，可以借给你抄一下，但它不属于你，抄完你还要还给我。（友情提示：非紧急情况不要抄作业）

另外还需要注意，我的作业可以借给你抄，但是你不能改我写的作业，我本来写对了你给我改错了，以后我还怎么借给你？所以，在calculate_length中，s是不可以修改的。

#### 可修改引用

如果我发现我写错了，让你帮我改一下怎么办？我授权给你，让你帮忙修改，你也需要表示能帮我修改就可以了。Rust也有办法。还记得我们前面介绍的可变变量和不可变变量吗？引用也是类似，我们可以使用mut关键字使引用可修改。

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

这样，我们就能在函数中对引用的值进行修改了。不过这里还要注意一点，在同一作用域内，对于同一个值，只能有一个可修改的引用。这也是因为Rust不想有并发修改数据的情况出现。

如果需要使用多个可修改引用，我们可以自己创建新的作用域：

```rust
let mut s = String::from("hello");

{
    let r1 = &mut s;

} // r1 超出作用域

let r2 = &mut s;
```

另一个冲突就是“读写冲突”，即不可变引用和可变引用之间的限制。

```rust
let mut s = String::from("hello");

let r1 = &s; // no problem
let r2 = &s; // no problem
let r3 = &mut s; // BIG PROBLEM

println!("{}, {}, and {}", r1, r2, r3);
```

这样的代码在编译时也会报错。这是因为不可变引用不希望在被使用之前，其指向的值被修改。这里只要稍微处理一下就可以了：

```rust
let mut s = String::from("hello");

let r1 = &s; // no problem
let r2 = &s; // no problem
println!("{} and {}", r1, r2);
// r1 和 r2 不再使用

let r3 = &mut s; // no problem
println!("{}", r3);
```

Rust编译器会在第一个print语句之后判断出r1和r2不会再被使用，此时r3还没有创建，它们的作用域不会有交集。所以这段代码是合法的。

### 空指针

对于可操作指针的编程语言来讲，最令人头疼的问题也许就是空指针了。通常情况是，在回收内存以后，又使用了指向这块内存的指针。而Rust的编译器帮助我们避免了这个问题（再次感谢Rust编译器）。

```rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String {
    let s = String::from("hello");

    &s
}
```

来看一下上面这个例子。在dangle函数中，返回值是字符串s的引用。但是在函数结束时，s的内存已经被回收了。所以s的引用就成了空指针。此时就会报expected lifetime parameter的编译错误。

### 另一种引用：Slice

除了引用之外，还有另一种没有ownership的数据类型叫做Slice。Slice是一种使用集合中一段序列的引用。

这里通过一个简单的例子来说明Slice的使用方法。假设我们需要得到给你字符串中的第一个单词。你会怎么做？其实很简单，遍历每个字符，如果遇到空格，就返回之前遍历过的字符的集合。

对字符串的遍历方法我来剧透一下，as_bytes函数可以把字符串分解成字节数组，iter是返回集合中每个元素的方法，enumerate是提取这些元素，并且返回(元素位置,元素值)这样的二元组的方法。这样是不是可以写出来了。

```rust
fn first_word(s: &String) -> usize {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return i;
        }
    }

    s.len()
}
```

来，感受下这个例子，虽然它返回的是第一个空格的位置，但是只要会字符串截取，还是可以达到目的的。不过不能剧透字符串截取了，不然暴露不出问题。

这么写的问题在哪呢？来看一下main函数。

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s);

    s.clear();
}
```

这里在获取空格位置后，对字符串s做了一个clear操作，也就是把s清空了。但word仍然是5，此时我们再去对截取s的前5个字符就会出问题。可能有人认为自己不会这么蠢，但是你愿意相信你的好（zhu）伙（dui）伴（you）也不会这么做吗？我是不相信的。那怎么办呢？这时候slice就要登场了。

使用slice可以获取字符串的一段字符序列。例如`&s[0..5]`可以获取字符串s的前5个字符。其中0为起始字符的位置下标，5是结束字符位置的下标加1。也就是说slice的区间是一个左闭右开区间。

slice还有一些规则：

- 如果起始位置是0，则可以省略。也就是说`&s[0..2]`和`&s[..2]`等价
- 如果起始位置是集合序列末尾位置，也可以省略。即`&s[3..len]`和`&s[3..]`等价
- 根据以上两条，我们还可以得出`&s[0..len]`和`&s[..]`等价

这里需要注意的是，我们截取字符串时，其边界必须是UTF-8字符。

有了slice，就可以解决我们的问题了

```rust
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

现在我们在main函数中对s执行clear操作时，编译器就不同意了。没错，又是万能的编译器。

除了slice除了可以作用于字符串以外，还可以作用于其他集合，例如：

```rust
let a = [1, 2, 3, 4, 5];

let slice = &a[1..3];
```

关于集合，我们以后会有更加详细的介绍。

### 总结

本文介绍的Ownership特性对于理解Rust来讲非常重要。我们介绍了什么是Ownership，Ownership的转移，以及不占用Ownership的数据类型Reference和Slice。

怎么样？是不是感觉今天的坑非常给力？如果之前在地下一层的话，那现在已经到地下三层了。所以请各位注意安全，有序降落。

