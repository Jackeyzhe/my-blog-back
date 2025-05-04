---
title: Redis命令详解：Sorted Sets
date: 2019-01-06 19:41:10
tags: Redis命令
---

Sorted Set（也称ZSET）和Set一样也是string类型的集合，你可以将它理解为Java中SortedSet和HashMap的集合体，一方面它是一个set，保证了元素的唯一性，另一方面它给每个value赋予了一个权重score，用来进行排序。集合中成员的最大个数为2<sup>32</sup>-1个。<!-- more -->

#### BZPOPMAX

最早可用版本：5.0.0

时间复杂度：O(log(N))，N是元素个数

用法：BZPOPMAX key [key ...] timeout

BZPOPMAX是ZPOPMAX的原始阻塞版。如果没有存在sorted set不能pop出元素，则连接会被阻塞。该命令会返回第一个非空的有序set的最高分的元素。

timeout参数是用来指定最大的阻塞时间，如果是0，则无限阻塞。

当没有元素被pop出，并且阻塞时间达到timeout时，返回nil。

如果有元素被pop出，则返回三个值：第一个是该元素来自哪个zset，第二个是pop元素的score，第三个是pop元素的value。

#### BZPOPMIN

最早可用版本：5.0.0

时间复杂度：O(log(N))，N是元素个数

用法：BZPOPMIN key [key ...] timeout

BZPOPMIN是ZPOPMIN的阻塞版本。它与BZPOPMAX相似，唯一不同的是它返回的是第一个非空有序set的最低分的元素。

#### ZADD

最早可用版本：1.2.0

时间复杂度：O(log(N))，N是元素个数

用法：ZADD key \[NX|XX\]\[CH\]\[INCR\]score member \[score member ...\]

将所有指定的成员和它的score加入zset，如果要插入的成员已经存在，则会更新该成员的分数，并将它排到正确的位置。如果key不存在，则创建一个新的zset并且插入成员。如果key存在，但不是zset类型，就会报错。score是双精度的浮点数，+inf和-inf同样有效。

在Redis3.2版本之后，ZADD命令支持了以下参数：

- XX：只更新已有的成员，不新增
- NX：只新增成员，不更新
- CH：将返回值从新增成员数修改为发生变化的成员总数
- INCR：当指定这个参数时，ZADD命令和ZINCRBY相似，但是只能接受一个成员的参数

##### 分数的范围

Redis的Sorted Set的分数范围从-(2^53)到+(2^53)。或者说是-9007199254740992 到 9007199254740992。更大的整数在内部用指数表示。

##### 相同分数的成员

由于所有的成员都是唯一的，当分数相同时，成员将按照字典序进行排序。它比较的是成员的字节数组，当所有成员的分数都相同时，范围查询可以用ZRANGEBYLEX命令（分数范围查询用ZRANGEBYSCORE命令）。

该命令返回值是新增成员的数量，如果是INCR参数模式，就返回新增成员的分数。

Redis2.4版本以后该命令才支持指定多个成员/分数对。

#### ZCARD

最早可用版本：1.2.0

时间复杂度：O(1)

当key存在时，返回zset的成员数量；否则返回0。

#### ZCOUNT

最早可用版本：2.0.0

时间复杂度：O(log(N))，N是zset的成员个数

用法：ZCOUNT key min max

返回分数在min到max（默认包括min和max）之间的成员个数。

ZCOUNT命令的时间复杂度为O(log(N))，因为它使用了ZRANK进行排序，然后获取范围的成员个数。

#### ZINCRBY

最早可用版本：1.2.0

时间复杂度：O(log(N))，N是zset的成员个数

用法：ZINCRBY key increment member

给指定zset中的指定的成员加上increment分数。如果成员不存在，则新增成员，将分数置为increment。如果key不存在，则先创建一个zset，然后加入新的成员。命令的返回值是成员的新分数。

#### ZINTERSTORE

最早可用版本：2.0.0

时间复杂度：O(N * K)+O(M * log(M))，N是输入的zset中的最小的成员数量，K为输入的zset的数量。M是结果中zset的成员数量

用法：ZINTERSTORE destination numkeys key \[key ...\]\[WEIGHTS weight \[weight ...\]]\[AGGREGATE SUM|MIN|MAX\]

ZINTERSTORE命令用于计算给出的numkeys个zset的交集，并将结果保存到destination中。在给出要计算的key和其他参数之前，必须先给出numkeys。默认情况下，输出的zset成员的分数，会是输入的zset的成员的分数之和。

``` bash
127.0.0.1:6379> ZADD myzset1 1 "jackey"
(integer) 1
127.0.0.1:6379> ZADD myzset1 2 "zhe"
(integer) 1
127.0.0.1:6379> ZADD myzset2 1 "jackey"
(integer) 1
127.0.0.1:6379> ZADD myzset2 2 "zhe"
(integer) 1
127.0.0.1:6379> ZADD myzset2 3 "2018"
(integer) 1
127.0.0.1:6379> ZINTERSTORE deszset 2 myzset1 myzset2
(integer) 2
127.0.0.1:6379> ZRANGE deszset 0 -1 WITHSCORES
1) "jackey"
2) "2"
3) "zhe"
4) "4"
```

WEIGHTS用来对每一个zset设置一个乘数因子，在计算分数时乘以指定的数值，默认是1。

AGGREGATE参数用来指定分数的聚合策略，默认是SUM，也就是相加。还可以选择取最大或最小的分数。

如果destination已经存在，则覆盖原来的值。命令的返回值是结果的成员个数。

#### ZLEXCOUNT

最早可用版本：2.8.9

时间复杂度：O(log(N))，N是zset的成员个数

用法：ZLEXCOUNT key min max

当所有成员的分数都相同时，使用这个命令计算min和max之间的成员个数。

关于min和max：

- 成员名称前需要加上[，[符号和成员名称之间不能有空格
- 可以使用-和+表示最大值和最小值
- 计算数量时，包括min和max

``` bash
127.0.0.1:6379> ZADD myzset 0 a 0 b 0 e 0 d 0 i 0 f 0 k
(integer) 7
127.0.0.1:6379> ZLEXCOUNT myzset - +
(integer) 7
127.0.0.1:6379> ZLEXCOUNT myzset b e
(error) ERR min or max not valid string range item
127.0.0.1:6379> ZLEXCOUNT myzset [b [e
(integer) 3
127.0.0.1:6379> ZRANGE myzset 0 -1
1) "a"
2) "b"
3) "d"
4) "e"
5) "f"
6) "i"
7) "k"
```

#### ZPOPMAX

最早可用版本：5.0.0

时间复杂度：O(log(N)*M)，N是zset的成员数量，M是弹出的成员数量

用法：ZPOPMAX key [count]

该命令用于移除并返回一定数量的分数最高的成员。count默认是1，count大于zset成员，当返回多个元素时，分数最高的最先被返回。

#### ZPOPMIN

最早可用版本：5.0.0

时间复杂度：O(log(N)*M)，N是zset的成员数量，M是弹出的成员数量

该命令和ZPOPMAX相反，返回的是分数最低的元素。只有这点不同，其他都相同。

#### ZRANGE

最早可用版本：1.2.0

时间复杂度：O(log(N)+M)，N是zset的成员数量，M是返回的成员数量

用法：ZRANGE key start stop [WITHSCORES]

该命令返回指定范围的成员，按照分数从低到高的顺序排。start和stop都是从0开始，也可以是负数，-1表示倒数第一个。返回的时候包括start和stop位置的成员。

如果start大于zset成员数量或者start大于stop，则返回空集合；如果stop大于最后一位，则返回start到最后一位的成员。

WITHSCORES参数表示返回的结果中是否要带分数。

#### ZRANGEBYLEX

最早可用版本：2.8.9

时间复杂度：O(log(N)+M)，N是zset的成员数量，M是返回的成员数量

用法：ZRANGEBYLEX key min max [LIMIT offset count]

前面我们提到过，当所有的成员的分数相同时，它们会按照字典顺序排列。对于中情况，ZRANGEBYLEX命令就是用来返回指定区间成员的。指定成员时可以使用(或者[，(表示不包含指定的成员，[表示包含。

成员字符串作为二进制数组来排序，默认是ASCII字符集的顺序。

LIMIT参数用于分页，类似于SQL中的LIMIT关键字。

#### ZRANGEBYSCORE

最早可用版本：1.0.5

时间复杂度：O(log(N)+M)，N是zset的成员数量，M是返回的成员数量

用法：ZRANGEBYSCORE key min max \[WITHSCORES\]\[LIMIT offset count\]

这个命令用来返回指定分数范围的成员，包括min和max。如果分数相同，则按字典顺序排列。

LIMIT参数用来分页。

在Redis2.0以后，可用使用WITHSCORES参数，使返回值中带有分数。

我们可以使用(表示不包括指定的分数，举个栗子：

``` bash
ZRANGEBYSCORE zset (1 5
```

取的分数范围是1<score<=5

#### ZRANK

最早可用版本：2.0.0

时间复杂度：O(log(N))

该命令用于返回指定的成员从低到高的排名。返回值从0开始，第一个元素的rank是0，第二个是1……

如果成员存在，返回它的rank值；如果不存在，返回nil。

#### ZREM

最早可用版本：1.2.0

时间复杂度：O(M*log(N))，N是zset的成员数量，M是要删除的成员数量

从zset中删除指定的成员。返回值为实际删除的成员数量。

Redis2.4版本以后支持一次指定多个成员。

#### ZREMRANGEBYLEX

最早可用版本：2.8.9

时间复杂度：O(log(N)+M)，N是zset的成员数量，M是要删除的成员数量

用法：ZREMRANGEBYLEX key min max

该命令用于删除指定返回的成员，最好用于所有分数都相同的集合，否则结果会不准确。

关于min和max的描述可以查看ZRANGEBYLEX命令。

#### ZREMRANGEBYRANK

最早可用版本：2.0.0

时间复杂度：O(log(N)+M)，N是zset的成员数量，M是要删除的成员数量

用法：ZREMRANGEBYRANK key start stop

用于删除指定rank范围的成员。start和stop的介绍可以查看ZRANGE命令。

#### ZREMRANGEBYSCORE

最早可用版本：1.2.0

时间复杂度：O(log(N)+M)，N是zset的成员数量，M是要删除的成员数量

用法：ZREMRANGEBYSCORE key min max

删除指定分数范围的成员，默认包括min和max的分数，在2.1.6版本以后可以不包括min和max，具体可以查看ZRANGEBYSCORE命令。

#### ZREVRANGE

最早可用版本：1.2.0

时间复杂度：O(log(N)+M)，N是zset的成员数量，M是要返回的成员数量

用法：ZREVRANGE key start stop [WITHSCORES]

返回分数从高到低的成员，也就是说，顺序与ZRANGE相反。其他条件都相同。

#### ZREVRANGEBYLEX

最早可用版本：1.2.0

时间复杂度：O(log(N)+M)，N是zset的成员数量，M是要返回的成员数量

该命令是ZRANGEBYLEX命令的倒序版本。

####ZREVRANGEBYSCORE

最早可用版本：2.2.0

时间复杂度：O(log(N)+M)，N是zset的成员数量，M是要返回的成员数量

是ZRANGEBYSCORE命令的倒序。

#### ZREVRANK

最早可用版本：2.0.0

时间复杂度：O(log(N))

是ZRANK的倒序。

#### ZSCAN

最早可用版本：2.8.0

时间复杂度：每次调用为O(1)

用法：ZSCAN key cursor \[MATCH pattern\]\[COUNT count\]

这是一个SCAN类的命令，可以看[这里](https://jackeyzhe.github.io/2018/09/26/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Redis%E7%9A%84scan%E5%91%BD%E4%BB%A4/)进行更深入的了解。

#### ZSCORE

最早可用版本：1.2.0

时间复杂度：O(1)

该命令用于返回指定成员的分数。如果指定成员不存在或者key不存在，则返回nil。

#### ZUNIONSTORE

最早可用版本：2.0.0

时间复杂度：O(N)+O(M log(M))，N是输入的zset的大小之和，M是结果的zset的大小

用法：ZUNIONSTORE destination numkeys key \[key ...\]\[WEIGHTS weight \[weight ...\]\]\[AGGREGATE SUM|MIN|MAX\]

计算给出的zset的并集，并把结果存到destination，在给定要计算的key和其他参数之前，要给出numkeys，也就是key的数量。默认情况下，结果中的成员的分数，是输入的zset的该成员分数的和。

关于WEIGHTS和AGGREGATE参数，可以查看ZINTERSTORE命令中的介绍。