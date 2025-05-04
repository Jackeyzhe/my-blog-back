---
title: Redis命令详解：Sets
date: 2018-12-19 23:47:02
tags: Redis命令
---

Redis的Set结构相当于Java中的HashSet，是无序的元素集合，并且元素都是唯一的。由于Set是通过hash表实现的，所以它的增加、删除、查找操作的时间复杂度都是O(1)。最大成员个数为2<sup>32</sup>-1。<!-- more -->

#### SADD

最早可用版本：1.0.0

时间复杂度：每个元素的添加的时间复杂度为O(1)，如果要添加N个，时间复杂度就为O(N)

用法：SADD key member [member...]

将指定的成员保存到key，如果成员已经存在，则直接忽略。如果key不存在，则先新建一个空set，再将成员添加进去。如果key存储不是一个set，则会报错。该命令执行成功后会返回实际添加成功的元素的个数。

在2.4版本之后可以支持多个参数，即一个命令添加多个成员。

#### SCARD

最早可用版本：1.0.0

时间复杂度：O(1)

返回key存储的set的元素个数。

#### SDIFF

最早可用版本：1.0.0

时间复杂度：O(N)，N是所给出的元素个数的总和

返回第一个set与后面元素的差集。不存在的key都被当做空set处理。

栗子时间：

``` bash
127.0.0.1:6379> SADD key1 a
(integer) 1
127.0.0.1:6379> SADD key1 b
(integer) 1
127.0.0.1:6379> SADD key1 c
(integer) 1
127.0.0.1:6379> SADD key2 b
(integer) 1
127.0.0.1:6379> SADD key3 c
(integer) 1
127.0.0.1:6379> SADD key3 d
(integer) 1
127.0.0.1:6379> SDIFF key1 key2 key3
1) "a"
```

#### SDIFFSTORE

最早可用版本：1.0.0

时间复杂度：O(N)，N是所给出的元素个数的总和

用法：SDIFFSTORE destination key [key...]

这个命令和SDIFF命令的作用相同，但是不同的是，该命令不返回差集，而是将差集存储到destination，如果destination已经存在，就将覆盖旧值。该命令的返回值是差集中元素的个数。

#### SINTER

最早可用版本：1.0.0

时间复杂度：O(N*M)，N是最小set的元素个数，M是set的个数

返回给出的所有set的交集。我们沿用刚刚SDIFF命令中使用的三个key，给key2增加一个元素c，此时三个key存储的元素情况为

``` bash
key1={a,b,c}
key2={b,c}
key3={c,d}
```

``` bash
127.0.0.1:6379> SADD key2 c
(integer) 1
127.0.0.1:6379> SINTER key1 key2 key3
1) "c"
```

#### SINTERSTORE

最早可用版本：1.0.0

时间复杂度：O(N*M)，N是最小set的元素个数，M是set的个数

该命令与SINTER的关系就像SDIFF与SDIFFSTORE的关系一样，因此我们不过多介绍了。

#### SISMEMBER

最早可用版本：1.0.0

时间复杂度：O(1)

该命令用于判断某个元素是否属于指定的key，如果属于，返回1；如果不属于或者key不存在，返回0。

``` bash
127.0.0.1:6379> SADD myset "jackeyzhe"
(integer) 1
127.0.0.1:6379> SISMEMBER myset "jackeyzhe"
(integer) 1
127.0.0.1:6379> SISMEMBER myset "2018"
(integer) 0
```

#### SMEMBERS

最早可用版本：1.0.0

时间复杂度：O(N)，N是set的元素个数

返回指定set的全部成员，当SINTER只有一个参数时，作用与该命令相同。

#### SMOVE

最早可用版本：1.0.0

时间复杂度：O(1)

将成员从一个set转移到另一个set中，这个操作是原子操作。如果源set不存在，或者不包含要转移的成员，那么就不会有任何操作，直接返回0。如果转移的成员在目标set中已经存在，那么只需要将该成员从源set中删除即可。如果源set或者目标set中的一个不是set结构，那么该命令就会报错。

如果成员被成功转移，就会返回1，如果没有进行转移操作，就会返回0。

```bash
127.0.0.1:6379> SADD from_set "a"
(integer) 1
127.0.0.1:6379> SADD from_set "b"
(integer) 1
127.0.0.1:6379> SMOVE from_set to_set "a"
(integer) 1
127.0.0.1:6379> SMEMBERS from_set
1) "b"
127.0.0.1:6379> SMEMBERS to_set
1) "a"
```

#### SPOP

最早可用版本：1.0.0

时间复杂度：O(1)

用法：SPOP key [count]

从指定set中删除并返回一个或多个随机元素。3.2版本以后支持count参数，即可以一次返回多个元素。如果key不存在，则返回nil。

如果count大于set中元素的个数，那么该命令就会返回set中现有的所有元素。

#### SRANDMEMBER

最早可用版本：1.0.0

时间复杂度：当没有count参数时是O(1)，否则为O(N)，N为count的绝对值

该命令用于随机返回set中的元素。从2.6版本开始支持count参数，如果count是正数，则返回count个不同元素的数组；如果count是负数，则允许同一个元素多次返回。

#### SREM

最早可用版本：1.0.0

时间复杂度：O(N)，N为指定member的个数

该命令用于从set中删除指定元素，如果不包含该元素，那么直接忽略。如果key不存在，则会当做空set处理，直接返回0。从2.4版本开始，该命令支持一次删除多个成员。

#### SSCAN

此命令是SCAN命令的同类，可以通过我的另一篇文章[深入理解Redis的scan命令](https://jackeyzhe.github.io/2018/09/26/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Redis%E7%9A%84scan%E5%91%BD%E4%BB%A4/)来进行更深入的了解

#### SUNION

最早可用版本：1.0.0

时间复杂度：O(N)，N为给出的所有set的元素个数之和

返回给定set的并集。

``` bash
127.0.0.1:6379> SMEMBERS key2
1) "b"
2) "c"
127.0.0.1:6379> SMEMBERS key3
1) "d"
2) "c"
127.0.0.1:6379> SUNION key2 key3
1) "d"
2) "b"
3) "c"
```

#### SUNIONSTORE

最早可用版本：1.0.0

时间复杂度：O(N)，N为给出的所有set的元素个数之和

该命令与SUNION的关系就像SDIFF与SDIFFSTORE的关系一样。

``` bash
127.0.0.1:6379> SUNIONSTORE mykey key2 key3
(integer) 3
127.0.0.1:6379> SMEMBERS mykey
1) "d"
2) "b"
3) "c"
```

