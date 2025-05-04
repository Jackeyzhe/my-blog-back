---
title: Rust入坑指南：居安思危
date: 2020-03-31 18:17:48
tags: Rust
---

任何事情都是相对的，就像Rust给我们的印象一直是安全、快速，但实际上，完全的安全是不可能实现的。因此，Rust中也是会有不安全的代码的。<!-- more -->

严格来讲，Rust语言可以分为**Safe Rust**和**Unsafe Rust**。Unsafe Rust是Safe Rust的超集。在Unsafe Rust中并不会禁用任何的安全检查，Unsafe Rust出现的原因是为了让开发者可以做一些更加底层的操作。这些事情本身也是不安全的，如果仍然要进行Rust的安全检查，那么就无法进行这些操作。

在进行下面这5种操作时，Unsafe Rust不会进行安全检查。

- 解引用原生指针
- 调用unsafe的函数或方法
- 访问或修改可变的静态变量
- 实现unsafe的trait
- 读写联合体中的字段

### 基础语法

Unsafe Rust的关键字是unsafe，它可以用来修饰函数、方法和trait，也可以用来标记代码块。

标准库中也有不少函数是unsafe的。例如String中的`from_utf8_unchecked()`函数。它的定义如下：

``` rust
pub unsafe fn from_utf8_unchecked(bytes: Vec<u8>) -> String {
  String { vec: bytes }
}
```

这个函数被标记为unsafe的原因是函数并没有检查传入参数是否是合法的UTF-8序列。也就是提醒使用者注意，使用这个函数要自己保证参数的合法性。

用unsafe标记的trait也比较常见，在前面我们见过的Send和Sync都是unsafe的trait。它们被用来保证线程安全， 将其标记为unsafe是告诉开发者，如果自己实现这两个trait，那么代码就会有安全风险。

我们在调用unsafe函数或方法时，需要使用unsafe代码块。

``` rust
fn main() {
    let sparkle_heart = vec![240, 159, 146, 150];
    
    let sparkle_heart = unsafe {
        String::from_utf8_unchecked(sparkle_heart)
    };

    assert_eq!("💖", sparkle_heart);
}
```

在了解了unsafe的基础语法之后，我们再来具体看看前面提到的5种操作。

### 解引用原生指针

Rust的原生指针分为两种：可变类型`*mut T`和不可变类型`*const T`。

与引用和智能指针不同，原生指针具有以下特性：

- 可以不遵循借用规则，在同一代码块中可以同时出现可变和不可变指针，也可以同时有多个可变指针
- 不保证指向有效内存
- 允许是null
- 不会自动清理内存

由这些特性可以看出，原生指针并不受Rust那一套安全规则的限制，因此，解引用原生指针是一种不安全的操作。换句话说，我们应该把这种操作放在unsafe代码块中。下面这段代码就展示了原生指针的第一条特性，以及如何解引用原生指针。

``` rust
fn main() {
    let mut num = 5;

    let r1 = &num as *const i32;
    let r2 = &mut num as *mut i32;

    unsafe {
        println!("r1 is: {}", *r1);
        println!("r2 is: {}", *r2);
    }
}
```

在Rust编程中，原生指针常被用作和C语言打交道，原生指针有一些特有的方法，例如可以用`is_null()`来判断原生指针是否是空指针，用`offset()`来获取指定偏移量的内存地址的内容，使用`read()/write()`方法来读写内存等。

### 调用unsafe的函数或方法

调用unsafe的函数或方法必须放到unsafe代码块中，这点我们在基础知识中已经介绍过。因为函数本身被标记为unsafe，也就意味着调用它可能存在风险。这点无需赘述。

### 访问或修改可变的静态变量

对于不可变的静态变量，我们访问它不会存在任何安全问题，但是对于可变的静态变量而言，如果我们在多线程中都访问同一个变量，那么就会造成数据竞争。这当然也是一种不安全的操作。所以要放到unsafe代码块中，此时线程安全应由开发者自己来保证。

``` rust
static mut COUNTER: u32 = 0;

fn add_to_count(inc: u32) {
    unsafe {
        COUNTER += inc;
    }
}

fn main() {
    add_to_count(3);

    unsafe {
        println!("COUNTER: {}", COUNTER);
    }
}
```

在这个例子中我们没有使用多线程，这里只是想展示一下如何访问和修改可变静态变量。

### 实现unsafe的trait

当trait中包含一个或多个编译器无法验证其安全性的方法时，这个trait就必须被标记为unsafe。而想要实现unsafe的trait，首先在实现代码块的关键字`impl`前也要加上unsafe标记。其次，无法被编译器验证安全性的方法，其安全性必须由开发者自己来保证。

前面我们也提到了，常见的unsafe的trait有Send和Sync这两个。

### 读写联合体中的字段

Rust中的Union联合体和Enum相似。我们可以使用union关键字来定义一个联合体。

``` rust
union MyUnion {
    i: i32,
    f: f32,
}
fn main() {
    let my_union = MyUnion{i: 3};
    unsafe {
        println!("{}", my_union.i);
    }
}
```

在初始化时，我们每次只能指定一个字段的值。这就造成我们在访问联合体中的字段时，有可能会访问到未定义的字段。因此，Rust让我们把访问操作放到unsafe代码块中，以此来警示我们必须自己保证程序的安全性。

### 总结

本文我们聊了Unsafe Rust的一些使用场景和使用方法。你只需要记住Unsafe的5种操作就好，在遇到这些操作时，一定要使用unsafe代码块。unsafe代码块不光是为了“骗”过编译器，要时刻提醒自己，**unsafe代码块中的程序要由开发者自己保证其正确性**。

- 解引用原生指针
- 调用unsafe的函数或方法
- 访问或修改可变的静态变量
- 实现unsafe的trait
- 读写联合体中的字段