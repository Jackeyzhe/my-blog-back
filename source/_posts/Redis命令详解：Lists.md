---
title: Redis命令详解：Lists
date: 2018-11-23 23:21:06
tags: Redis命令
---

List是Redis的基础数据类型之一，类似于Java中的LinkedList。一个列表最多包含2<sup>32</sup>个元素，常被用作模拟队列操作，接下来我们具体介绍一下List相关的命令。<!-- more -->



#### BLPOP

最早可用版本：2.0.0

时间复杂度：O(1)

用法：

``` bash
BLPOP key [key ...] timeout
```

BLPOP是LPOP的阻塞版本，当列表没有元素可以被弹出时，连接将被阻塞。当给定多个key，会按参数key的顺序检查各个列表，弹出第一个非空列表的的头元素。timeout表示阻塞的最大秒数，timeout为0表示无限阻塞。

这里有一个问题，当多个元素同时push进一个list时，阻塞的BLPOP命令会有什么操作。在说明之前，我们先思考一下如何操作才会出现这样的情况：

- 对list执行LPUSH mylist a b c这样的命令
- 对同一个list进行多次push操作，这些操作是在事务中执行的
- 使用Redis2.6以后的版本执行Lua脚本进行push操作

对于这个问题，Redis2.4版本和Redis2.6以后的版本处理方法有所不同。

假如客户端A执行命令

``` bash
BLPOP mylist 0
```

这时mylist为空，客户端A会被阻塞，此时客户端B执行了命令

``` bash
LPUSH mylist a b c
```

如果在Redis2.6版本之后，客户端A会返回c，因为在客户端Bpush了元素a、b、c后，其从左到右的顺序是c、b、a，但是在Redis2.4版本中，客户端会在push操作的上下文，所以当LPUSH开始往list里push第一个元素时，它就被传送到客户端A，也就是客户端A会接收到a。

有时，我们会有这样的需求：我们需要为了等待Set的新元素而阻塞队列，这样就需要一个阻塞版的SPOP，可惜目前还没有支持这样的命令。不过我们可以使用BLPOP命令来实现，下面是实现的伪代码：

消费者：

``` bash
LOOP forever
    WHILE SPOP(key) returns elements
        ... process elements ...
    END
    BRPOP helper_key
END
```

生产者：

``` bash
MULTI
SADD key element
LPUSH helper_key x
EXEC
```



#### BRPOP

最早可用版本：2.0.0

时间复杂度：O(1)

它与BLPOP基本相同，不同的地方在于它是从尾部弹出元素，而BLPOP是从头部弹出元素。



#### BRPOPLPUSH

最早可用版本：2.2.0

时间复杂度：O(1)

用法：

``` bash
BRPOPLPUSH source destination timeout
```

它是RPOPLPUSH的阻塞版本，当source包含元素时，它与RPOPLPUSH表现的一样，当source为空时，Redis会被阻塞，直到另一个客户端push元素，或者达到timeout时间限制。



#### LINDEX

最早可用版本：1.0.0

时间复杂度：O(N)，N是找到目标元素所跨越元素的个数，当目标元素为第一个或者最后一个时，时间复杂度为O(1)。

该命令用于返回列表中指定位置的元素，index是从0开始的，-1表示倒数第一个元素，-2表示倒数第二个元素，以此类推。当key不是一个list时，会返回一个错误。当index超出范围时返回nil。



#### LINSERT

最早可用版本：2.2.0

时间复杂度：O(N)，N为在找到基准value前所跨越的元素个数。也就是说，如果插入到头部，时间复杂度为O(1)，如果插入到尾部，时间复杂度为O(N)。

用法：

``` bash
LINSERT key BEFORE|AFTER pivot value
```

该命令把value插入到基准值pivot的前面或者后面，如果key不存在，list被当做空列表，不会发生任何操作。如果key存储的不是list，则会报错。命令的返回值是，插入操作后，list的长度，如果找不到基准值pivot，则会返回-1。



#### LLEN

最早可用版本：1.0.0

时间复杂度：O(1)

返回指定key的list的长度，如果key不存在，则被看作是空列表，返回0。如果key存储的不是list，则会报错。



#### LPOP

最早可用版本：1.0.0

时间复杂度：O(1)

该命令用于删除并返回list的第一个元素。当key不存在时，返回nil。



#### LPUSH

最早可用版本：1.0.0

时间复杂度：O(1)

将所有指定的value插入列表的头部，如果key不存在，就先创建一个空列表并进行插入操作，如果key存储的不是list，则会返回一个错误。我们可以一次插入多个元素，他们从左到右依次被插入到list中，因此，

``` bash
LPUSH mylist a b c
```

 命令生成的列表，c是第一个元素，a是第三个元素。该命令的返回值是插入操作后列表的长度。需要注意的一点是：在Redis2.4版本以前（不包括2.4）是不支持一次插入多个元素的。



#### LPUSHX

最早可用版本：2.2.0

时间复杂度：O(1)

当key存在时，在头部插入指定元素，key不存在时，不进行插入操作。



#### LRANGE

最早可用版本：1.0.0

时间复杂度：O(S+N)，S是start元素的偏移量，N是指定范围元素的个数

用法：

``` bash
LRANGE key start stop
```

返回指定key的指定范围的元素，start和stop都是下标（从0开始），同样，下标可以是负数，-1表示倒数第一个，-2表示倒数第二个。命令返回的结果会包含下标为stop的元素。如果start超出list的长度返回，则会返回一个空的列表，如果stop超出list的长度返回，则会返回到最后一个元素。



#### LREM

最早可用版本：1.0.0

时间复杂度：O(1)

用法：

``` bash
LREM key count value
```

移除list中前count次出现的value

- count>0时：从头到尾匹配value
- count=0时：移除全部匹配到value的元素
- count<0时，从尾部到头部匹配value

当key不存在时，被当做空列表看待，直接返回0。



#### LSET

最早可用版本：1.0.0

时间复杂度：O(N)，N为list的长度

设置指定下标的value，如果下标超出范围，则会返回一个错误。



#### LTRIM

最早可用版本：1.0.0

时间复杂度：O(N)，N删除掉的元素的个数

该命令用来修剪一个已经存在的list，修剪后的list只包含指定范围的元素。start和stop都是从0开始的索引，例如，

``` bash
LTRIM foobar 0 2
```

就是只保留foobar的前3个元素。start和stop也可以是负数，-1表示倒数第一个元素，-2表示倒数第二个，以此类推。如果下标超出范围，并不会报错，而是进行如下处理：如果start比list的最后一个元素的下标大，或者start>end，结果就是空list，如果end大于最大下标，Redis会将其当成最后一个元素来处理。



#### RPOP

最早可用版本：1.0.0

时间复杂度：O(1)

删除并返回list的最后一个元素。当key不存在时，返回nil。



#### RPOPLPUSH

最早可用版本：1.2.0

时间复杂度：O(1)

原子性的返回并删除source的最后一个元素，并把该元素存储到destination的第一个元素的位置。举个栗子，source保存了元素a、b、c，destination保存了x、y、z，执行了

``` bash
RPOPLPUSH source destination
```

后，source保存的会是a、b，而destination保存的则是c、x、y、z。该命令的返回值是那个从source被移出和存入destination的元素。



#### RPUSH

最早可用版本：1.0.0

时间复杂度：O(1)

将指定元素插入到指定key的尾部。如果key不存在，就创建一个空的列表。如果key保存的不是list，则会返回一个错误。在2.4版本之后，可以使用一条命令一次插入多个值，插入的顺序是从左到右。



#### RPUSHX

最早可用版本：2.2.0

时间复杂度：O(1)

它和RPUSH唯一不同的一点就是如果key不存在，就不会进行任何操作。