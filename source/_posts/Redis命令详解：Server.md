---
title: Redis命令详解：Server
date: 2019-07-01 22:29:21
tags: Redis命令
---

Redis命令学习，服务器篇<!-- more -->

#### BGREWRITEAOF

最早可用版本：1.0.0

使Redis重写AOF文件，重写后的AOF文件相较于当前版本的AOF文件占用的空间更小。即使重写失败，数据也不会丢失，因为在重写成功前，旧版本的AOF文件不会改动。重写操作只会在后台没有其他持久化工作时进行：

- 如果Redis子进程正在保存快照，那么重写AOF的操作会到保存工作完成后才开始进行。这种情况下，该命令仍然会返回OK，但是会增加一条额外的返回信息说明。在Redis2.6以后的版本，你可以使用INFO命令查看重写操作是否被预定执行。
- 如果已经有一个重写AOF命令正在进行，那么该命令会报错，并且不会预定执行重写操作。

Redis2.4版本以后，重写AOF操作会自动触发。想要了解更多信息可以查看[持久化文档](https://redis.io/topics/persistence)。

#### BGSAVE

最早可用版本：1.0.0

在后台保存当前数据库到磁盘。命令会马上返回OK，Redis会fork出一个子进程来进行此操作，而父进程继续提供服务。可以使用LASTSAVE命令查看保存操作是否成功。

#### CLIENT GETNAME

最早可用版本：2.6.9

时间复杂度：O(1)

这个命令会返回当前连接使用CLIENT SETNAME设置的连接名称，如果没有设置，则返回空。

#### CLIENT ID

最早可用版本：5.0.0

时间复杂度：O(1)

返回当前连接的ID。每个连接都会保证两点：

1. 不会重复，所以如果返回的ID相同，那么调用方就可以确定底层是没有断开重连的。
2. ID单调递增，如果一个连接的ID大于另一个连接的ID，那么它一定晚于这个连接创建。

``` bash
jackeyzhe@ubuntu:~/redis-5.0.4/src$ ./redis-cli 
127.0.0.1:6379> CLIENT ID
(integer) 3
127.0.0.1:6379> 
jackeyzhe@ubuntu:~/redis-5.0.4/src$ ./redis-cli 
127.0.0.1:6379> CLIENT ID
(integer) 4
```

#### CLIENT KILL

最早可用版本：2.4.0

时间复杂度：O(N)，N是客户端连接数

用法：CLIENT KILL \[ip:port\] \[ID client-id\]\[TYPE normal|master|slave|pubsub\] \[ADDR ip:port\]\[SKIPME yes/no\]

这个命令用来关闭一个指定的客户端连接。在Redis2.8.11之前，都可以指定要关闭的连接地址，像下面这种形式：

``` bash
CLIENT KILL addr:port
```

ip:port应该和CLIENT LIST命令中的一行匹配。

在2.8.12及以后的版本，则可以使用以下形式：

``` bash
CLIENT KILL <filter> <value> ... ... <filter> <value>
```

这种形式支持多种根据多种属性匹配客户端：

- CLIENT KILL ADDR ip:port ：这种和旧的形式相同
- CLIENT KILL ID client-id ：这种形式允许关闭指定ID的连接
- CLIENT KILL TYPE type ：这种形式支持关闭某种类型的客户端，type取值为：normal, master, slave和pubsub（master在Redis3.2之后可以使用）
- CLIENT KILL SKIPME yes/no ： 参数默认是yes，也就是不会关闭发出命令的客户端，而如果指定为no，则连自己也一起关闭

**注意：从Redis5开始type不再使用slave，改为replica**

上述的多种过滤器也可以组合使用。使用新的形式时，返回值为关闭的客户端数量。由于Redis是单线程的，所以这个命令不能关闭一个正在执行命令的客户端。

#### CLIENT LIST

最早可用版本：2.4.0

时间复杂度：O(N)，N是客户端连接数

用法：CLIENT LIST [TYPE normal|master|replica|pubsub]

这个命令用来查看连接的客户端信息，在Redis5之后，可以使用TYPE参数。

``` bash
127.0.0.1:6379> CLIENT LIST
id=3 addr=127.0.0.1:44994 fd=8 name= age=342 idle=3 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=keys
id=4 addr=127.0.0.1:44996 fd=9 name= age=335 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=26 qbuf-free=32742 obl=0 oll=0 omem=0 events=r cmd=client
```

返回值：每行代表一个客户端连接，字段包括：

- id：一个64bit唯一ID
- name：使用CLIENT SETNAME设置的客户端名称
- addr：客户端的地址和端口号
- fd：相应的socket文件描述符
- age：连接时长，单位为秒
- idle：空闲时间，单位为秒
- flags：客户端标志
- db：当前数据库ID
- sub：已订阅频道的数量
- psub：已订阅模式的数量
- multi：事务中的命令数
- qbuf：查询缓存的长度
- qbuf-free：查询缓存空闲空间（0表示缓存已满）
- obl：输出缓存的长度
- oll ：输出列表长度（缓存满时，回复会被放入这个列表中）
- omem：输出缓存的内存占用量
- events：文件描述符事件
- cmd：最后一次执行的命令

客户端标志包括以下几种：

- A：尽可能快的关闭连接
- b：客户端在等待阻塞时间
- c：写完回复之后关闭连接
- d：被监视的key被修改了，事务将失败
- i：客户端正在等待虚拟机I/O（已废弃）
- M：客户端是master节点
- N：没有设置flag
- O：客户端是MONITOR模式
- P：客户端是Pub/Sub的订阅者
- r：客户端是针对集群节点的只读模式
- S：客户端连接到此实例的从节点
- u：客户端未阻塞
- U：客户端通过Unix套接字连接
- x：客户端正在执行事务

文件描述符事件包括：

r：客户端套接字可读

w：客户端套接字可写

#### CLIENT PAUSE

最早可用版本：2.9.50

时间复杂度：O(1)

这个命令可以使所有连接暂停一段时间（单位：毫秒）。这个命令通常用来将连接从一个Redis实例迁移到另一个实例，例如当一个实例需要进行系统升级时，我们应该这样做：

1. 使用CLIENT PAUSE暂停所有客户端
2. 等待几秒钟，以便从节点与主节点数据同步完成
3. 将一个从节点切换成主节点
4. 重新使客户端连接到新的主节点

这个命令通常在事务中和INFO replication命令一起使用，这样做可以使从节点和主节点同步完成。

#### CLIENT REPLY

最早可用版本：3.2

时间复杂度：O(1)

这个命令用来禁止服务器对当前客户端回复。它有以下几种使用场景：

1. 客户端发送fire和forget命令时（不关心什么时候完成的命令）
2. 加载大量数据
3. 正在创建缓存

在这些情况下，客户端会忽略服务器的回复，因此，服务器回复是一种资源的浪费。

命令支持3个参数：

- ON：默认，接收服务器所有回复
- OFF：不接收服务器的所有回复
- SKIP：不接收下一条命令的回复

#### CLIENT SETNAME

最早可用版本：2.6.9

时间复杂度：O(1)

这个命令用来给连接设置一个名字。这个命令会在CLIENT LIST的输出列表中显示。名字的长度没有限制，但一般不超过Redis字符串类型的长度（512MB）。名字里不能有空格。可以通过设置空字符串的方式来删除一个连接的名称，每个新的连接是没有名称的。

#### CLIENT UNBLOCK

最早可用版本：5.0.0

时间复杂度：O(log N) N是客户端连接数

用法：CLIENT UNBLOCK client-id [TIMEOUT|ERROR]

这个命令可以解除被阻塞的客户端（执行了BPOP、XREAD、WAIT等命令）。

默认情况下，如果阻塞超时，会解除阻塞。这里也可以有其他参数，TIMEOUT或ERROR。如果设置为ERROR，那么，被强制解除阻塞的连接会返回一个-UNBLOCKED错误。

这个命令主要用于少量连接监控多个key时，如果要监控新的key，又不想使用更多的连接，那么就解除一个连接的阻塞，监控新的key后再重新阻塞。

#### COMMAND

最早可用版本：2.8.13

时间复杂度：O(N)，N是Redis命令总数

返回所有Redis命令的相关信息。

返回信息的第一层包含以下内容：

- 命令的名称
- 命令arity（可接受的参数数量）
- 命令标志
- 第一个key在参数列表中的位置
- 最后一个key在参数列表中的位置
- 用于定位重复key的step

命令arity如果是整数，表示命令的请求参数（包括命令名称）数量是一个固定的值；如果是负数，表示请求参数的最小数量。

``` bash
1) 1) "get"
   2) (integer) 2
   3) 1) readonly
   4) (integer) 1
   5) (integer) 1
   6) (integer) 1
```

``` bash
1) 1) "mget"
   2) (integer) -2
   3) 1) readonly
   4) (integer) 1
   5) (integer) -1
   6) (integer) 1
```

命令标志包括以下几种：

- write：命令会改变数据
- readonly：命令不会改变key的值
- denyoom：如果发生OOM，则拒绝命令
- admin：服务器管理员命令
- pubsub：和订阅模式有关的命令
- noscript：脚本中不能执行的命令
- random：命令的执行结果随机
- sort_for_scrpt：如果在脚本中执行，结果会被排序
- loading：允许命令在数据库加载时执行
- stale：副本中有过时数据时，仍然可以执行命令
- skip_monitor：不在MONITOR中显示命令
- asking：集群相关，导入时仍可执行命令
- fast：命令操作时间不变或者是log(N)
- movablekeys：命令没有预先执行的key，必须自己指定

#### COMMAND COUNT

最早可用版本：2.8.13

时间复杂度：O(1)

返回当前Redis服务器支持的命令数量

#### COMMANC GETKEYS

最早可用版本：2.8.13

时间复杂度：O(N)

输出命令中的key

``` bash
> COMMAND GETKEYS mset a b c d e f
1) "a"
2) "c"
3) "e"
```

#### COMMAND INFO

最早可用版本：2.8.13

时间复杂度：O(N)

返回指定命令的详细信息，返回结果的内容和COMMAND一样，如果命令不存在，返回nil。

``` bash
> COMMAND INFO get
1) 1) "get"
   2) (integer) 2
   3) 1) readonly
      2) fast
   4) (integer) 1
   5) (integer) 1
   6) (integer) 1
```

#### CONFIG GET

最早可用版本：2.0.0

这个命令可以读redis服务器的配置参数，在2.6版本以后，才可以读到全部配置。命令支持模糊匹配

``` bash
config get *max-*-entries*
1) "hash-max-zipmap-entries"
2) "512"
3) "list-max-ziplist-entries"
4) "512"
5) "set-max-intset-entries"
6) "512"
```

#### CONFIG RESETSTAT

最早可用版本：2.0.0

时间复杂度：O(1)

重置INFO命令中的一些统计信息，包括

- Key命中数
- Key未命中数
- 命令处理数量
- 连接数
- 过期Key的数量
- 拒绝的连接数
- 最近的fork(2)时间
- `aof_delayed_fsync` 计数器

#### CONFIG REWRITE

最早可用版本：2.8.0

该命令用于重写redis.conf文件，应用最小的改变，使其反映当前服务器的配置。如果原始文件不存在，该命令也可以重头写一个配置文件。

#### CONFIG SET

最早可用版本：2.0.0

该命令用于修改服务器的配置。可以使用`CONFIG GET *`查看可修改的配置。

#### DBSIZE

最早可用版本：1.0.0

返回当前数据库key的数量

#### DEBUG OBJECT

最早可用版本：1.0.0

这个命令不应该在客户端使用，具体请看[OBJECT](https://redis.io/commands/object)命令。

#### DEBUG SEGFAULT

最早可用版本：1.0.0

这个命令用于执行无效的内存访问，导致Redis崩溃，它用于在开发过程中模拟错误。

#### FLUSHALL

最早可用版本：1.0.0

删除所有数据库中的key。

4.0.0版本以后，可以使用ASYNC参数，这个参数可以在后台进行删除任务。

#### FLUSHDB

最早可用版本：1.0.0

删除当前数据库的所有key。

#### INFO

INFO命令返回服务器的详细信息。可以执行显示的部分：

- server：Redis server通用信息
- clients：客户端连接部分
- memory：内存相关信息
- persistence：RDB和AOF相关信息
- stats：通用统计信息
- replication：主从复制信息
- cpu：CPU相关统计
- commandstats：Redis命令统计
- cluster：Redis集群部分
- keyspace：数据库相关信息

#### LASTSAVE

最早可用版本：1.0.0

返回DB最后一次保存成功的时间。

#### MEMORY DOCTOR

最早可用版本：4.0.0

该命令报告Redis服务器遇到的与内存相关的问题，并就可能的补救措施提出建议。

#### MEMORY HELP

最早可用版本：4.0.0

该命令返回描述不同子命令的帮助文本。

#### MEMORY MALLOC-STATS

最早可用版本：4.0.0

该命令提供了内存分配器的内部统计报告。这个命令只有在使用jemalloc作为分配器时可用。

#### MEMORY PURGE

最早可用版本：4.0.0

该命令尝试清除脏页面，以便内存分配器回收。

#### MEMORY STATS

最早可用版本：4.0.0

返回内存的使用情况，包括以下维度：（没有特别说明，则以字节为单位）

- peak.allocated：Redis内存消耗的峰值
- total.allocated：Redis使用的内存总数
- startup.allocated：Redis启动时，初始化所需要的内存
- replication.backlog：复制log积压的大小
- clients.slaves：所有副本的总开销
- clients.normal：所有客户端的总开销
- aof.buffer：当前AOF缓冲区的总开销
- dbXXX：对于每个数据库，主字典和到期字典的开销
- overhead.total：所有的间接开销
- keys.count：所有数据库中key的总数
- keys.bytes-per-key：净内存使用和keys.count的比率
- dataset.bytes：数据集的开销
- dataset.percentage：数据集开销所占百分比
- peak.percentage：peak.allocated占total.allocated的百分比
- fragmentation：碎片内存的比率

#### MEMORY USAGE

最早可用版本：4.0.0

时间复杂度：O(N)

用法 `MEMORY USAGE key [Samples count]`

该命令返回了指定key和它的value存储所占用的内存大小。

对于嵌套数据类型，可以使用SAMPLES参数，其中count是采样嵌套的数量，默认是5，如果要对所有嵌套值进行采样，需要将SAMPLES设置为0。

#### MONITOR

最早可用版本：1.0.0

MONITOR是一个调试命令，它可以回溯Redis服务器处理的每个命令。它可以帮助理解数据库发生了什么。这个命令可以通过redis-cli和telnet使用。

安全起见，某些命令是不会被MONITOR记录的（如CONFIG）

#### REPLICAOF

最早可用版本：5.0.0

这个命令可以改变从服务器的从属关系。

对于一台从服务器来说，执行`REPLICAOF NO ONE`命令，结果是当前服务器变成master。而执行`REPLICAOF host port`命令会改变原从属关系，是从服务器归属于新的master。

#### ROLE

最早可用版本：2.8.12

返回Redis实例的角色信息：包括：master、slave和sentinel

对于master节点：

``` bash
1) "master"
2) (integer) 3129659
3) 1) 1) "127.0.0.1"
      2) "9001"
      3) "3129242"
   2) 1) "127.0.0.1"
      2) "9002"
      3) "3129543"
```

第一行是master字符串；第二行是主从复制的偏移量；用于标记重新同步时开始的位置，第三行开始是从节点的信息，包括IP、端口号和最后同步的从节点偏移量。

对于从节点：

``` bash
1) "slave"
2) "127.0.0.1"
3) (integer) 9000
4) "connected"
5) (integer) 3167038
```

第一行返回slave字符串；第二行是IP；第三行是端口号；第四行是与主节点连接状态，可以是connect（需要与主节点连接），connecting（正在连接），sync（尝试进行主从同步），connected（从节点在线）；第五行是从节点收到的数据量

对于sentinel

``` bash
1) "sentinel"
2) 1) "resque-master"
   2) "html-fragments-master"
   3) "stats-master"
   4) "metadata-master"
```

第一行是sentinel；第二行之后是监控master的名字。

#### SAVE

最早可用版本：1.0.0

同步的执行保存当前数据集快照，并写入到RDB文件。**不要在生产环境使用这个命令！**

#### SHUTDOWN

最早可用版本：1.0.0

这个命令有以下操作：

- 停止全部客户端
- 如果设置了save point，就会执行SAVE命令
- 如果AOF是enabled，刷新AOF文件
- 退出服务器

如果启用了持久化，则可以保证数据不丢失。

如果执行SHUTDOWN SAVE，即便没有save point，仍然会强制执行保存操作。

如果执行SHUTDOWN NOSAVE，有保存点也不会执行保存操作。

#### SLAVEOF

最早可用版本：1.0.0

该命令被REPLICAOF替代

#### SLOWLOG

最早可用版本：2.2.12

这个命令用来读取并重置慢查询的日志。通过*slowlog-log-slower-than*参数设置慢查询的时间，超过这个时间就会被记录

#### TIME

最早可用版本：2.6.0

时间复杂度：O(1)

该命令返回当前服务器时间的秒数，以及当前秒中已经过去的微秒数。