---
title: Redis命令详解：HyperLogLog
date: 2019-01-15 00:13:03
tags: Redis
---

HyperLogLog是Redis的高级数据结构，它在做基数统计的时候非常有用，每个HyperLogLog的键可以计算接近2<sup>64</sup>不同元素的基数，而大小只需要12KB。<!-- more -->

HyperLogLog目前只支持3个命令，PFADD、PFCOUNT、PFMERGE。我们先来逐一介绍一下。

#### PFADD

最早可用版本：2.8.9

时间复杂度：O(1)

将参数中的元素都加入指定的HyperLogLog数据结构中，这个命令会影响基数的计算。如果执行命令之后，基数估计改变了，就返回1；否则返回0。如果指定的key不存在，那么就创建一个空的HyperLogLog数据结构。该命令也支持不指定元素而只指定键值，如果不存在，则会创建一个新的HyperLogLog数据结构，并且返回1；否则返回0。

#### PFCOUNT

最早可用版本：2.8.9

时间复杂度：O(1)，对于多个比较大的key的时间复杂度是O(N)

对于单个key，该命令返回的是指定key的近似基数，如果变量不存在，则返回0。

对于多个key，返回的是多个HyperLogLog并集的近似基数，它是通过将多个HyperLogLog合并为一个临时的HyperLogLog，然后计算出来的。

HyperLogLog可以用很少的内存来存储集合的唯一元素。（每个HyperLogLog只有12K加上key本身的几个字节）

HyperLogLog的结果并不精准，错误率大概在0.81%。

需要注意的是：该命令会改变HyperLogLog，因此使用8个字节来存储上一次计算的基数。所以，从技术角度来讲，PFCOUNT是一个写命令。

##### 性能问题

即使理论上处理一个存储密度大的HyperLogLog需要花费较长时间，但是当指定一个key时，PFCOUNT命令仍然具有很高的性能。这是因为PFCOUNT会缓存上一次结算的基数，而多数PFADD命令不会更新寄存器。所以才可以达到每秒上百次请求的效果。

当处理多个key时，最耗时的一步是合并操作。而通过计算出来的并集的基数是不能缓存的。所以多个key的处理速度一般在毫秒级。

#### PFMERGE

最早可用版本：2.8.9

时间复杂度：O(N)，N是要合并的HyperLogLog的数量

用法：PFMERGE destkey sourcekey [sourcekey ...]

合并多个HyperLogLog，合并后的基数近似于合并前的基数的并集（observed Sets）。计算完之后，将结果保存到指定的key。

除了这三个命令，我们还可以像操作String类型的数据那样，对HyperLogLog数据使用SET和GET命令。关于HyperLogLog的原理以及其他细节，我将在后面介绍，敬请期待。