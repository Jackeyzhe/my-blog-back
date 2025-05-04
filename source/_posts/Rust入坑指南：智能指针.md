---
title: Rust入坑指南：智能指针
date: 2020-03-09 22:26:45
tags: Rust
---

在了解了Rust中的所有权、所有权借用、生命周期这些概念后，相信各位坑友对Rust已经有了比较深刻的认识了，今天又是一个连环坑，我们一起来把智能指针刨出来，一探究竟。<!-- more -->

智能指针是Rust中一种特殊的数据结构。它与普通指针的本质区别在于普通指针是对值的借用，而智能指针通常拥有对数据的所有权。在Rust中，如果你想要在堆内存中定义一个对象，并不是像Java中那样直接new一个，也不是像C语言中那样需要手动malloc函数来分配内存空间。Rust中使用的是`Box::new`来对数据进行封箱，而`Box<T>`就是我们今天要介绍的智能指针之一。除了`Box<T>`之外，Rust标准库中提供的智能指针还有`Rc<T>`、`Ref<T>`、`RefCell<T>`等等。在详细介绍之前，我们还是先了解一下智能指针的基本概念。

### 基本概念

我们说Rust的智能指针是一种特殊的数据结构，那么它特殊在哪呢？它与普通数据结构的区别在于智能指针实现了`Deref`和`Drop`这两个traits。实现`Deref`可以使智能指针能够解引用，而实现`Drop`则使智能指针具有自动析构的能力。

#### Deref

Deref有一个特性是强制隐式转换：**如果一个类型T实现了Deref<Target=U>，则该类型T的引用在应用的时候会被自动转换为类型U**。

``` rust
use std::rc::Rc;
fn main() {
    let x = Rc::new("hello");
    println!("{:?}", x.chars());
}
```

如果你查看Rc的源码，会发现它并没有实现chars()方法，但我们上面这段代码却可以直接调用，这是因为Rc实现了Deref。

``` rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<T: ?Sized> Deref for Rc<T> {
    type Target = T;

    #[inline(always)]
    fn deref(&self) -> &T {
        &self.inner().value
    }
}
```

这就使得智能指针在使用时被自动解引用，像是不存在一样。

Deref的内部实现是这样的：

``` rust
#[lang = "deref"]
#[doc(alias = "*")]
#[doc(alias = "&*")]
#[stable(feature = "rust1", since = "1.0.0")]
pub trait Deref {
    /// The resulting type after dereferencing.
    #[stable(feature = "rust1", since = "1.0.0")]
    type Target: ?Sized;

    /// Dereferences the value.
    #[must_use]
    #[stable(feature = "rust1", since = "1.0.0")]
    fn deref(&self) -> &Self::Target;
}

#[lang = "deref_mut"]
#[doc(alias = "*")]
#[stable(feature = "rust1", since = "1.0.0")]
pub trait DerefMut: Deref {
    /// Mutably dereferences the value.
    #[stable(feature = "rust1", since = "1.0.0")]
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

DerefMut和Deref类似，只不过它是返回可变引用的。

#### Drop

Drop对于智能指针非常重要，它是在智能指针被丢弃时自动执行一些清理工作，这里所说的清理工作并不仅限于释放堆内存，还包括一些释放文件和网络连接等工作。之前我总是把Drop理解成Java中的GC，随着对它的深入了解后，我发现它比GC要强大许多。

Drop的内部实现是这样的：

``` rust
#[lang = "drop"]
#[stable(feature = "rust1", since = "1.0.0")]
pub trait Drop {
    #[stable(feature = "rust1", since = "1.0.0")]
    fn drop(&mut self);
}
```

这里只有一个drop方法，实现了Drop的结构体，在消亡之前，都会调用drop方法。

``` rust
use std::ops::Drop;
#[derive(Debug)]
struct S(i32);

impl Drop for S {
    fn drop(&mut self) {
        println!("drop {}", self.0);
    }
}

fn main() {
    let x = S(1);
    println!("create x: {:?}", x);
    {
        let y = S(2);
        println!("create y: {:?}", y);
    }
}
```

上面代码的执行结果为

![结果](https://res.cloudinary.com/dxydgihag/image/upload/v1583204435/Blog/rust/11/rust11-1.png)

可以看到x和y在生命周期结束时都去执行了drop方法。

对智能指针的基本概念就先介绍到这里，下面我们进入正题，具体来看看每个智能指针都有什么特点吧。

### Box<T>

前面我们已经提到了Box<T>在Rust中是用来在堆内存中保存数据使用的。它的使用方法非常简单：

``` rust
fn main() {
    let x = Box::new("hello");
    println!("{:?}", x.chars())
}
```

我们可以看一下`Box::new`的源码

``` rust
#[stable(feature = "rust1", since = "1.0.0")]
#[inline(always)]
pub fn new(x: T) -> Box<T> {
  box x
}
```

可以看到这里只有一个box关键字，这个关键字是用来进行堆内存分配的，它只能在Rust源码内部使用。box关键字会调用Rust内部的exchange_malloc和box_free方法来管理内存。

``` rust
#[cfg(not(test))]
#[lang = "exchange_malloc"]
#[inline]
unsafe fn exchange_malloc(size: usize, align: usize) -> *mut u8 {
    if size == 0 {
        align as *mut u8
    } else {
        let layout = Layout::from_size_align_unchecked(size, align);
        let ptr = alloc(layout);
        if !ptr.is_null() {
            ptr
        } else {
            handle_alloc_error(layout)
        }
    }
}

#[cfg_attr(not(test), lang = "box_free")]
#[inline]
pub(crate) unsafe fn box_free<T: ?Sized>(ptr: Unique<T>) {
    let ptr = ptr.as_ptr();
    let size = size_of_val(&*ptr);
    let align = min_align_of_val(&*ptr);
    // We do not allocate for Box<T> when T is ZST, so deallocation is also not necessary.
    if size != 0 {
        let layout = Layout::from_size_align_unchecked(size, align);
        dealloc(ptr as *mut u8, layout);
    }
}
```

### Rc<T>

在前面的学习中，我们知道Rust中一个值在同一时间只能有一个变量拥有其所有权，但有时我们可能会需要多个变量拥有所有权，例如在图结构中，两个图可能对同一条边拥有所有权。

对于这样的情况，Rust为我们提供了智能指针Rc<T>（reference counting）来解决共享所有权的问题。每当我们通过Rc共享一个所有权时，引用计数就会加一。当引用计数为0时，该值才会被析构。

Rc<T>是单线程引用计数指针，不是线程安全类型。

我们还是通过一个简单的例子来看一下Rc<T>的应用吧。（示例来自[the book](https://doc.rust-lang.org/book/ch15-04-rc.html)）

如果我们想要造一个“双头”的链表，如下图所示，3和4都指向5。我们先来尝试使用Box实现。

![双头链表](https://res.cloudinary.com/dxydgihag/image/upload/v1583289009/Blog/rust/11/rust11-2.svg)

``` rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use crate::List::{Cons, Nil};

fn main() {
    let a = Cons(5,
                 Box::new(Cons(10,
                               Box::new(Nil))));
    let b = Cons(3, Box::new(a));
    let c = Cons(4, Box::new(a));
}
```

上述代码在编译时就会报错，因为a绑定给了b以后就无法再绑定给c了。

![Box无法共享所有权](https://res.cloudinary.com/dxydgihag/image/upload/v1583289016/Blog/rust/11/rust11-3.png)

``` rust
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a));
    let c = Cons(4, Rc::clone(&a));
    println!("count a {}", Rc::strong_count(&a));
}
```

这时我们可以看到a的引用计数是3，这是因为这里计算的是节点5的引用计数，而a本身也是对5的一次绑定。这种通过clone方法共享所有权的引用称作**强引用**。

Rust还为我们提供了另一种智能指针Weak<T>，你可以把它当作是Rc<T>的另一个版本。它提供的引用属于**弱引用**。它共享的指针没有所有权。但他可以帮助我们有效的避免循环引用。

### RefCell<T>

前文中我们聊过变量的可变性和不可变性，主要是针对变量的。按照前面所讲的，对于结构体来说，我们也只能控制它的整个实例是否可变。实例的具体某个成员是否可变我们是控制不了的。但在实际开发中，这样的场景也是比较常见的。比如我们有一个User结构体：

``` rust
struct User {
    id: i32,
    name: str,
    age: u8,
}
```

通常情况下，我们只能修改一个人的名称或者年龄，而不能修改用户的id。如果我们把User的实例设置成了可变状态，那就不能保证别人不会去修改id。

为了应对这种情况，Rust为我们提供了`Cell<T>`和`RefCell<T>`。它们本质上不属于智能指针，而是可以提供内部可变性的容器。内部可变性实际上是一种设计模式，它的内部是通过一些`unsafe`代码来实现的。

我们先来看一下`Cell<T>`的使用方法吧。

``` rust
use std::cell::Cell;
struct Foo {
    x: u32,
    y: Cell<u32>,
}

fn main() {
    let foo = Foo { x: 1, y: Cell::new(3)};
    assert_eq!(1, foo.x);
    assert_eq!(3, foo.y.get());
    foo.y.set(5);
    assert_eq!(5, foo.y.get());
}
```

我们可以使用Cell的set/get方法来设置/获取起内部的值。这有点像我们在Java实体类中的setter/getter方法。这里有一点需要注意：`Cell<T>`中包裹的T必须要实现Copy才能够使用get方法，如果没有实现Copy，则需要使用Cell提供的get_mut方法来返回可变借用，而set方法在任何情况下都可以使用。由此可见Cell并没有违反借用规则。

对于没有实现Copy的类型，使用`Cell<T>`还是比较不方便的，还好Rust还提供了`RefCell<T>`。话不多说，我们直接来看代码。

``` rust
use std::cell::RefCell;
fn main() {
    let x = RefCell::new(vec![1, 2, 3]);
    println!("{:?}", x.borrow());
    x.borrow_mut().push(5);
    println!("{:?}", x.borrow());
}
```

从上面这段代码中我们可以观察到`RefCell<T>`的borrow_mut和borrow方法对应了`Cell<T>`中的set和get方法。

`RefCell<T>`和`Cell<T>`还有一点区别是：`Cell<T>`没有运行时开销（不过也不要用它包裹大的数据结构），而`RefCell<T>`是有运行时开销的，这是因为使用`RefCell<T>`时需要维护一个借用检查器，如果违反借用规则，则会引起线程恐慌。

### 总结

关于智能指针我们就先介绍这么多，现在我们简单总结一下。Rust的智能指针为我们提供了很多有用的功能，智能指针的一个特点就是实现了`Drop`和`Deref`这两个trait。其中`Drop`trait中提供了drop方法，在析构时会去调用。`Deref`trait提供了自动解引用的能力，让我们在使用智能指针的时候不需要再手动解引用了。

接着我们分别介绍了几种常见的智能指针。`Box<T>`可以帮助我们在堆内存中分配值，`Rc<T>`为我们提供了多次借用的能力。`RefCell<T>`使内部可变性成为现实。

最后再多说一点，其实我们以前见到过的`String`和`Vec`也属于智能指针。

至于它们为什么属于智能指针，Rust又提供了哪些其他的智能指针呢？这里就留个坑吧，感兴趣的同学可以自己踩一下。