---
title: Java原子操作类，知多少？
date: 2018-08-23 23:27:30
tags: Java
---

前文我们介绍了Java并发编程中的两个关键字：volatile和synchronized。我们也知道了volatile虽然是轻量级，但不能保证原子性，synchronized可以保证原子性，但是比较重量级。<!-- more -->

那么有没有一种简单的、性能高的方法来保证Java的原子操作呢？答案当然是有的，本文就为大家揭秘一些在JDK1.5时期加入Java家族的成员——Atomic包。Atomic包下包含了12个类，分为4种类型：

- 原子更新基本类型
- 原子更新数组
- 原子更新引用
- 原子更新字段

下面我来为大家一一引荐。



#### 原子基本类型

原子基本类型，从名称上就可以看出，是为基本类型提供原子操作的类。它们是以下3位：

- AtomicBoolean
- AtomicInteger
- AtomicLong

这三位属于近亲，提供的方法基本一模一样（AtomicBoolean支持方法略少）。

![AtomicBoolean](https://res.cloudinary.com/dxydgihag/image/upload/v1535039613/Blog/Atomic/AtomicBoolean.png)

![AtomicInteger](https://res.cloudinary.com/dxydgihag/image/upload/v1535039617/Blog/Atomic/AtomicInteger.png)

![AtomicLong](https://res.cloudinary.com/dxydgihag/image/upload/v1535039623/Blog/Atomic/AtomicLong.png)

这里我们以AtomicInteger为例介绍这些方法。

- void lazySet(int newValue)：使用此方法后最终会被设置成newValue。是线程不安全的。官方解释如下：

  > As probably the last little JSR166 follow-up for Mustang, we added a "lazySet" method to the Atomic classes (AtomicInteger, AtomicReference, etc). This is a niche method that is sometimes useful when fine-tuning code using non-blocking data structures. The semantics are that the write is guaranteed not to be re-ordered with any previous write, but may be reordered with subsequent operations (or equivalently, might not be visible to other threads) until some other volatile write or synchronizing action occurs).
  >
  > The main use case is for nulling out fields of nodes in non-blocking data structures solely for the sake of avoiding long-term garbage retention; it applies when it is harmless if other threads see non-null values for a while, but you'd like to ensure that structures are eventually GCable. In such cases, you can get better performance by avoiding the costs of the null volatile-write. There are a few other use cases along these lines for non-reference-based atomics as well, so the method is supported across all of the AtomicX classes.
  >
  > For people who like to think of these operations in terms of machine-level barriers on common multiprocessors, lazySet provides a preceeding store-store barrier (which is either a no-op or very cheap on current platforms), but no store-load barrier (which is usually the expensive part of a volatile-write).

  这里解释道：此方法不可与之前的写操作进行重排序，可以与之后的写操作进行重排序，知道出现volatile写或synchronizing操作。好处是比普通的set方法性能要好，前提是可以忍受其他线程在一段时间内读到的是旧数据。

- int getAndSet(int newValue)：以原子方式更新，并且返回旧值。

- boolean compareAndSet(int expect, int update)：如果输入的值等于expect的值，则以原子方式更新。

- int getAndIncrement()：以原子方式自增，返回的是自增前的值。

- int getAndDecrement()：与getAndIncrement相反，返回的是自减前的值。

- int getAndAdd(int delta)：以原子方式，将当前值与输入值相加，返回的是计算前的值。

- int incrementAndGet()：以原子方式自增，返回自增后的值。

- int decrementAndGet()：以原子方式自减，返回自减后的值。

- int addAndGet(int delta)：以原子方式，将当前值与输入值相加，返回的是计算后的值。

- int getAndUpdate(IntUnaryOperator updateFunction)：Java1.8新增方法，以原子方式，按照指定方法更新当前数值，返回更新前的值，需要注意的是，提供的方法应该无副作用（side-effect-free），即两次执行结果相同，原因是如果由于线程争用导致更新失败会尝试再次执行该方法。

- int updateAndGet(IntUnaryOperator updateFunction)：同样是Java1.8新增方法，与getAndUpdate唯一不同的是返回值是更新后的值。

- int getAndAccumulate(int x, IntBinaryOperator accumulatorFunction)：与上述两个方法类似，操作数由参数x提供。返回更新前的值。

- int accumulateAndGet(int x, IntBinaryOperator accumulatorFunction)：与getAndAccumulate方法作用相同，返回更新后的值。



方法介绍完了，AtomicInteger是怎么实现原子操作的呢？一起来看一下getAndIncrement方法的源码。

``` java
/**
 * Atomically increments by one the current value.
 *
 * @return the previous value
 */
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
```

继续看Unsafe方法里的getAndIncrement方法

``` java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

其中getIntVolatile方法是一个本地方法，根据对象以及偏移量获取对应的值。然后执行compareAndSwapInt方法，该方法根据对象和偏移量获取当当前值，与希望的值var5比较，如果相等，则将值更新为var5+var4。否则，进入循环。如果想要了解UnSafe类的其他方法，可以阅读源码或者参考这篇文章[[Java中Unsafe类详解](https://www.cnblogs.com/mickole/articles/3757278.html)](http://www.cnblogs.com/mickole/articles/3757278.html)。



#### 原子数组

下面的类是为数组中某个元素的更新提供原子操作的类。

- AtomicIntegerArray
- AtomicLongArray
- AtomicReferenceArray

这三个类中的方法也都是类似的：

![AtomicIntegerArray](https://res.cloudinary.com/dxydgihag/image/upload/v1535123459/Blog/Atomic/AtomicIntegerArray.png)

![AtomicLongArray](https://res.cloudinary.com/dxydgihag/image/upload/v1535123464/Blog/Atomic/AtomicLongArray.png)

![AtomicReferenceArray](https://res.cloudinary.com/dxydgihag/image/upload/v1535123468/Blog/Atomic/AtomicReferenceArray.png)

我们对AtomicIntegerArray中的方法进行介绍。

- AtomicIntegerArray(int length)：构造函数，新建一个数组，传入AtomicIntegerArray。
- AtomicIntegerArray(int[] array)：构造函数，将array克隆一份，传入AtomicIntegerArray，因此，修改AtomicIntegerArray中的元素时不会影响原数组。
- int length()：获取数组长度。
- int get(int i)：获取位置i的元素。
- void set(int i, int newValue)：设置对应位置的值。
- void lazySet(int i, int newValue)：类似AtomicInteger中的lazySet。
- int getAndSet(int i, int newValue)：更新对应位置的值，返回更新前的值。
- boolean compareAndSet(int i, int expect, int update)：比较对应位置的值与期望值，如果相等，则更新，返回true。如果不能返回false。
- int getAndIncrement(int i)：对位置i的元素以原子方式自增，返回更新前的值。
- int getAndDecrement(int i)：对位置i的元素以原子方式自减，返回更新前的值。
- int getAndAdd(int i, int delta)：对位置i的元素以原子方式计算，返回更新前的值。
- int incrementAndGet(int i)、int decrementAndGet(int i)、addAndGet(int i, int delta)：这三个方法与上面三个方法操作相同，区别是这三个方法返回的是更新后的值。

下面四个方法都是1.8才加入的，根据提供的参数中的方法对位置i的元素进行操作。区别是返回值不同以及是否提供操作数。

- int getAndUpdate(int i, IntUnaryOperator updateFunction)
- int updateAndGet(int i, IntUnaryOperator updateFunction)
- int getAndAccumulate(int i, int x, IntBinaryOperator accumulatorFunction)
- int accumulateAndGet(int i, int x, IntBinaryOperator accumulatorFunction)

原子数组类型同样也是调用Unsafe类的方法，因此原理与基本类型的原理相同，这里不做赘述。



#### 原子引用类型

前面讲到的类型都只能以原子的方式更新一个变量，有没有办法以原子方式更新多个变量呢？我们可以利用了面向对象的封装思想，可以把多个变量封装成一个类，再以原子的方式更新一个类对象。幸运的是，Atomic为我们提供了更新引用类型的方法。一起来认识一下他们吧。

- AtomicReference
- AtomicReferenceFieldUpdater
- AtomicMarkableReference

同样的，先来看一下这三个类提供的方法有哪些。

![AtomicReference](https://res.cloudinary.com/dxydgihag/image/upload/v1535125768/Blog/Atomic/AtomicReference.png)

![AtomicReferenceFieldUpdater](https://res.cloudinary.com/dxydgihag/image/upload/v1535125769/Blog/Atomic/AtomicReferenceFieldUpdater.png)

![AtomicStampedReference](https://res.cloudinary.com/dxydgihag/image/upload/v1535126255/Blog/Atomic/AtomicMarkableReference.png)

方法的作用与AtomicInteger中的方法类似，不做过多介绍。

``` java
/**
 * Atomically sets the value to the given updated value
 * if the current value {@code ==} the expected value.
 * @param expect the expected value
 * @param update the new value
 * @return {@code true} if successful. False return indicates that
 * the actual value was not equal to the expected value.
 */
public final boolean compareAndSet(V expect, V update) {
    return unsafe.compareAndSwapObject(this, valueOffset, expect, update);
}
```

这是compareAndSet方法的源码，同样是调用UnSafe类的CAS方法，因此，原子操作的原理也和基本类型相同。



#### 原子更新字段类

前文提到了AtomicReferenceFieldUpdater类，它更新的是类的字段，除了这个类，Atomic还提供了另外三个类用于更新类中的字段：

- AtomicIntegerFieldUpdater
- AtomicLongFieldUpdater
- AtomicStampedReference

使用这些类时需要注意以下几点：

1. 更新字段必须有volatile关键字修饰
2. 更新字段不能是类变量
3. 使用前需要调用newUpdater()方法创建一个Updater

这三个类的方法语义也很明确，可以参考AtomicInteger。

![AtomicIntegerFieldUpdater](https://res.cloudinary.com/dxydgihag/image/upload/v1535127319/Blog/Atomic/AtomicIntegerFieldUpdater.png)

![AtomicLongFieldUpdater](https://res.cloudinary.com/dxydgihag/image/upload/v1535127323/Blog/Atomic/AtomicLongFieldUpdater.png)

![AtomicStampedReference](https://res.cloudinary.com/dxydgihag/image/upload/v1535125773/Blog/Atomic/AtomicStampedReference.png)



#### 总结

Atomic包提供了足够的原子类供我们使用，想要真正完全理解这些类，还需要不断的练习。