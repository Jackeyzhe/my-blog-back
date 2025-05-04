---
title: Redis命令详解：Streams
date: 2019-07-01 23:13:32
tags: Redis命令
---

Redis5.0迎来了一种新的数据结构Streams，没有了解过的同学可以先阅读[前文](https://jackeyzhe.github.io/2019/03/28/%E3%80%90%E8%AF%91%E3%80%91Redis%E5%96%9C%E6%8F%90%E6%96%B0%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%EF%BC%9ARedis-streams/)，今天来介绍一下Streams相关的命令。<!-- more -->

#### XACK

最早可用版本：5.0.0

时间复杂度：O(1)

用法：XACK key group ID [ID...]

这个命令用于删除消费者组的pending entries list中的元素。通常情况下，调用XREADGROUP命令或者消费者调用XCLAIM命令时，会使一个消息阻塞，并存到PEL中，阻塞的消息被发送给消费者时，服务器并不知道消息是否被处理。

当消费者成功消费消息后，会调用XACK命令，服务器就会将消息从PEL中删除，并释放内存。

#### XADD

最早可用版本：5.0.0

时间复杂度：O(1)

向指定的stream添加元素。如果key不存在，就创建一个新的stream。

entry由一系列field-value对组成，存储顺序由用户添加顺序决定。XADD命令是唯一一个向stream中添加数据的命令。删除数据的命令则有XDEL和XTRIM。

在stream中，entry ID是唯一标识。XADD命令中ID参数是*时，会自动生成唯一ID。然而在生产环境中并不常用，通常需要我们指定一种格式较好的唯一ID。

默认的ID生成策略是：“Unix毫秒时间戳-同一毫秒值内的序列号”。

当用户显式指定ID时，最小值是0-1，且ID必须是递增的。

用户可以使用MAXLEN指定stream的最大元素数量

``` bash
XADD mystream MAXLEN ~ 1000 * ... entry fields here ..
```

上面的波浪线表示不是严格的限制1000个，也可以多出几十个。

#### XCLAIM

最早可用版本：5.0.0

时间复杂度：O(log N)

用法：XCLAIM key group consumer min-idle-time ID \[ID ...\]\[IDLE ms\] \[TIME ms-unix-time\] \[RETRYCOUNT count\]\[FORCE\] \[JUSTID\]

这个命令用于改变pending消息的所有权，新的owner是命令参数中的consumer。

命令的使用场景是：

1. 一个消费者关联了一个stream
2. 消费者A通过XREADGROUP读取一条消息
3. 这个消息被加入到PEL中，并发送给指定的消费者，但是没有调用XACK命令来确认
4. 这时消费者突然挂掉
5. 其他的消费者就会使用XPENDING命令检查待处理消息列表，为了继续处理这些命令，它们使用XCLAIM命令改变这些消息的所有者。

接下来解释一下命令的各个选项：

1. IDLE<ms>：设置消息空闲时间，默认是0。消息只有在空闲时间大于IDLE时才会被认领。
2. TIME<ms-unix-time>：和IDLE相同，不过它是绝对时间
3. RETRYCOUNT <count>：设置重试次数，通常XCLAIM不会改变这个值，它通常用于XPENDING命令，用来发现一些长时间未被处理的消息。
4. FORCE：在PEL中创建待处理消息，即使指定的ID尚未分配给客户端的PEL。
5. JUSTID：只返回认领的消息ID数组，不返回实际消息。

#### XDEL

最早可用版本：5.0.0

时间复杂度：O(1)

删除stream中的entry并返回删除的数量。

#### XGROUP

最早可用版本：5.0.0

时间复杂度：每个子命令是O(1)

该命令用于管理stream相关的消费者组。使用XGROUP命令你可以：

- 创建与一个stream相关联的消费者组
- 销毁一个消费者组
- 从消费者组中删除指定的消费者
- 设置消费者组的last delivered ID

创建新的消费者组的命令是：

``` bash
XGROUP CREATE mystream consumer-group-name $
```

最后一个参数是stream中已传递的最后一个ID，使用$表示这个消费者组只能获取到新的元素。

销毁消费者组的命令是：

``` bash
XGROUP DESTROY mystream some-consumer-group
```

即使消费者组存在活跃的消费者和等待消息，它仍然会被删除，所以执行这个命令需要格外谨慎。

删除指定消费者的命令是：

``` bash
XGROUP DELCONSUMER mystream consumer-group-name myconsumer123
```

当一个新的consumer的名字被提到时，就会自动创建消费者。当消费者不再使用时，我们可以将它删除，上面的命令返回消费者在被删除之前所拥有的待处理消息。

设置last delivered ID的命令是：

``` bash
XGROUP SETID mystream my-consumer-group 0
```

最后，如果不记得语法，可以使用命令：

``` bash
XGROUP HELP
```

#### XINFO

最早可用版本：5.0.0

时间复杂度：O(N)，N是CONSUMERS和GROUPS返回的item数量

用法：XINFO \[CONSUMERS key groupname\] \[GROUPS key\]\[STREAM key\] \[HELP\]

这个命令用于返回stream和相关消费者组的不同信息。它有三种形式。

- XINFO STREAM <key>        这个命令返回stream的通用信息
- XINFO GROUPS <key>       这个命令用于获得stream相关的消费者组的信息
- XINFO CONSUMERS <key>  <group> 这个命令返回指定消费者组的消费者列表

#### XLEN

最早可用版本：5.0.0

时间复杂度：O(1)

返回stream中的entry数量。如果key不存在，则返回0。对于长度为0的stream，Redis不会删除，因为可能存在关联的消费者组。

#### XPENDING

最早可用版本：5.0.0

时间复杂度：O(N)，N是返回的元素数量

用法：XPENDING key group \[start end count\] \[consumer\]

通过消费者组捕获数据，但不是确认这些数据。

XPENDING命令是检查待处理消息列表的接口，用于观察和了解消费者组正在发生的事情：哪些客户端是活跃的，哪些消息等待消费，或者查看是否有空闲的消息。这个命令通常与XCLAIM一起使用，用于处理长时间未被处理的消息。

这个命令的返回值是：

``` bash
> XPENDING mystream group55 - + 10
1) 1) 1526984818136-0
   2) "consumer-123"
   3) (integer) 196415
   4) (integer) 1
```

其中包括：

1. 消息ID
2. 获取并要确认消息的消费者名称
3. 自上次消息传递给消费者以来经过的毫秒数
4. 该消息被传递的次数

#### XRANGE

最早可用版本：5.0.0

时间复杂度：O(N)，N是返回的元素数量

用法：XRANGE key start end [COUNT count]

该命令用于返回stream中指定ID范围的数据，可以使用-和+表示最小和最大ID。ID也可以指定为不完全ID，即只指定Unix时间戳，就可以获取指定时间范围内的数据。

#### XREAD

最早可用版本：5.0.0

时间复杂度：O(N)，N是返回的元素数量

用法：XREAD \[COUNT count\] \[BLOCK milliseconds\] STREAMS key \[key ...\] ID \[ID ...\]

从一个或多个stream中读取数据，仅返回ID大于调用者报告的最后接收ID的条目。

BLOCK项用于指定阻塞时长。STREAMS项必须在最后，用于指定stream和ID。

#### XREADGROUP

最早可用版本：5.0.0

时间复杂度：O(log(N)+M) ，N是返回的元素数量，M是一个常量。

用法：XREADGROUPGROUP group consumer \[COUNT count\] \[BLOCK milliseconds\] STREAMS key \[key ...\] ID \[ID ...\]

XREADGROUP是XREAD的特殊版本，支持消费者组。

#### XREVRANGE

最早可用版本：5.0.0

时间复杂度：O(log(N)+M) ，N是返回的元素数量，M是一个常量。

此命令与XRANGE唯一的区别是顺序相反。

#### XTRIM

最早可用版本：5.0.0

时间复杂度：O(log(N)+M) ，N是返回的元素数量，M是一个常量。

用法：XTRIM key MAXLEN [~] count

该命令用于裁剪流为指定数量的项目。这个命令被设计为接受多种策略，但目前只实现了MAXLEN一种。

如果要裁剪到stream中最新的1000个项目：

``` bash
XTRIM mystream MAXLEN 1000
```

可以使用以下形式提高效率：

``` bash
XTRIM mystream MAXLEN ~ 1000
```

~表示用户不需要精确的1000个项目，可以多出几十个，但是不能少于1000.