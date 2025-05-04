---
title: Redis命令详解：Connection
date: 2018-09-19 23:00:05
tags: Redis命令
---

最近在学习Redis的相关知识，上一篇我们也介绍了Redis的安装方法和基本数据结构，后面就打算开一个新的系列文章：Redis命令详解。既是对基础的巩固，也是为了以后查询起来更方便。<!-- more -->



整个系列会分为以下几个部分：

- Connection
- [Keys](https://jackeyzhe.github.io/2018/09/22/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9AKeys/)
- [Strings](https://jackeyzhe.github.io/2018/10/07/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9AStrings/)
- [Hashs](https://jackeyzhe.github.io/2018/11/22/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9AHashs/)
- [Lists](https://jackeyzhe.github.io/2018/11/23/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9ALists/)
- [Sets](https://jackeyzhe.github.io/2018/12/19/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9ASets/)
- [Sorted Sets](https://jackeyzhe.github.io/2019/01/06/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9ASorted-Sets/)
- [HyperLogLog](https://jackeyzhe.github.io/2019/01/15/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9AHyperLogLog/)
- [Transactions](https://jackeyzhe.github.io/2019/03/04/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9ATransactions/)
- [Server](https://jackeyzhe.github.io/2019/07/01/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9AServer/)
- [Streams](https://jackeyzhe.github.io/2019/07/01/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9AStreams/)
- [Pub/Sub](https://jackeyzhe.github.io/2019/08/28/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9APub-Sub/)
- [Cluster](https://jackeyzhe.github.io/2019/08/28/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9ACluster/)
- [Geo](https://jackeyzhe.github.io/2019/09/06/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9AGeo/)
- [Scripting](https://jackeyzhe.github.io/2019/06/10/Redis-Lua%E8%84%9A%E6%9C%AC%E4%B8%AD%E5%AD%A6%E6%95%99%E7%A8%8B%EF%BC%88%E4%B8%8A%EF%BC%89/)



今天我们从Redis连接的相关命令开始。



#### AUTH

可用版本：1.0.0

AUTH命令用于检测密码是否与配置文件中的密码是否一致，如果一致，则服务器会返回OK，并且继续接受后面的命令，否则，Redis会拒绝执行接下来的命令。

``` bash
127.0.0.1:6379> config set requirepass "mypass"
OK
127.0.0.1:6379> AUTH my
(error) ERR invalid password
127.0.0.1:6379> ping
(error) NOAUTH Authentication required.
127.0.0.1:6379> AUTH mypass
OK
127.0.0.1:6379> ping
PONG
```

需要注意的是：由于Redis的读写性能非常高，所以可以在段时间内处理许多次AUTH操作，这样使得密码被暴力破解的可能性增加，所以我们在设置密码的时候需要尽量使密码安全性更强。



#### ECHO

可用版本：1.0.0

ECHO命令打印字符串。

``` bash
127.0.0.1:6379> ECHO "Hello!"
"Hello!"
```



#### PING

可用版本：1.0.0

PING命令用于检测服务器是否在运行，或者测试延迟。正常情况下，如果没有参数，则服务器会返回一个PONG，如果有参数的话，服务器会将参数复制一份，返回为字符串。

``` bash
127.0.0.1:6379> PING
PONG
127.0.0.1:6379> PING "hi"
"hi"
```



#### QUIT

可用版本：1.0.0

QUIT命令用于关闭当前连接，当所有等待中的回复都写入客户端后，就会立即关闭当前连接。



#### SELECT

可用版本：1.0.0

SELECT命令用于切换数据库，参数为数据库索引号。一个新连接的默认数据库索引号是0，所有的数据库都持久化到一个相同的RDB或AOF文件。不同的数据库可以有相同的key。

``` bash
127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]>
```

切换数据库后，提示符后面会出现数据库索引号。需要注意的是：当使用Redis Cluster时，不能使用SELECT命令。



#### SWAPDB

可用版本：4.0.0

SWAPDB用于交换两个数据库，连接到这个数据库的其他客户端会立即看到另一个数据库的数据。

``` bash
#client 0
127.0.0.1:6379> set db db_0
OK
127.0.0.1:6379> get db
"db_0"
127.0.0.1:6379> SWAPDB 0 1
OK
127.0.0.1:6379> get db
(nil)

#client 1
127.0.0.1:6379> SELECT 1
OK
127.0.0.1:6379[1]> get db
"db_0"
```





