---
title: Redis命令详解：Transactions
date: 2019-03-04 23:29:54
tags: Redis命令
---

Redis的事务和我们常见的数据库的事务最大的区别就是，Redis的事务中如果有一个命令执行失败，其他命令仍然可以执行成功。Redis的事务以MULTI开始，由EXEC触发。在EXEC前的操作都将被放入缓存队列中。在事务执行过程中其他客户端的命令不会插到事务中执行。下面就来介绍一下Redis事务相关的命令。<!-- more -->

#### DISCARD

最早可用版本：2.0.0

放弃所有队列中的命令，将连接状态置为正常状态。如果事务被WATCH，则取消所有的WATCH。

#### EXEC

最早可用版本：1.2.0

执行队列中的全部命令，将连接状态置为正常状态。如果某些key处于被监视状态，并且队列中有和这些key相关的命令。那么EXEC命令只有在这些key的值没有变化的情况下事务才会执行，否则事务被打断。

#### MULTI

最早可用版本：1.2.0

标记事务块的开始，之后的命令被顺序插入缓存队列中，可以用EXEC命令执行这些命令。

#### UNWATCH

最早可用版本：2.2.0

时间复杂度：O(1)

清除掉所有被WATCH的key，如果调用了EXEC或者DISCARD命令，则不用手动调用UNWATCH命令。

#### WATCH

最早可用版本：2.2.0

时间复杂度：对每个都是O(1)

将指定的key标记为被监视状态，如果事务执行前被改动，则事务会被打断。

最后举一个事务被打断的栗子

``` bash
127.0.0.1:6379> SET lock_time 1
OK
127.0.0.1:6379> WATCH lock_time
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> SET transcation_key z #这时另一个客户端执行了命令 SET lock_time 2
QUEUED
127.0.0.1:6379> INCR lock_time
QUEUED
127.0.0.1:6379> EXEC
(nil)
127.0.0.1:6379> GET transcation_key
(nil)
```