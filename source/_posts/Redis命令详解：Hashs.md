---
title: Redis命令详解：Hashs
date: 2018-11-22 22:45:56
tags: Redis命令
---

Hash是一种String类型的field、value的映射表，因此，它非常适合存储对象。下面我们来一一介绍与Hash相关的命令。<!-- more -->



#### HDEL

最早可用版本：2.0.0

时间复杂度：O(N)，其中N为要删除的field的个数

HDEL命令用于删除指定key的指定的一个或多个field。如果指定的field不存在于指定的key中则会被忽略，如果指定的key不存在，会当做空的hash进行处理，向客户端返回0。

命令的返回值是实际删除的field的个数，不包括不存在的field。

从2.4.0版本开始，该命令支持一次删除多个field。在此之前，如果想一次性删除多个field，只能利用Redis的事务来实现。



#### HEXISTS

最早可用版本：2.0.0

时间复杂度：O(1)

HEXISTS命令用来验证指定的key是否包含指定的field，如果包含，返回1；如果不包含或者key不存在，返回0。



#### HGET

最早可用版本：2.0.0

时间复杂度：O(1)

返回指定的key中指定的field对应的value。如果field不在key中或者key不存在，则返回nil。



#### HGETALL

最早可用版本：2.0.0

时间复杂度：O(N)，N为hash的大小，即key中field的个数。

返回key所存储的所有field以及field对应的value。每个value跟在field的后面被返回，因此，返回值的长度是hash的size的2倍。如果key不存在，则返回空列表。

``` bash
127.0.0.1:6379> HGETALL noexist
(empty list or set)
127.0.0.1:6379> HSET mykey field1 "follow"
(integer) 1
127.0.0.1:6379> HSET mykey field2 "Jackeyzhe2018"
(integer) 1
127.0.0.1:6379> HGETALL mykey
1) "field1"
2) "follow"
3) "field2"
4) "Jackeyzhe2018"
```



#### HINCRBY

最早可用版本：2.0.0

时间复杂度：O(1)

用法：

``` bash
HINCRBY key field increment
```

用来对指定key的指定field进行增量操作，返回计算后的结果。如果key不存在，或者key中不包含指定的field，则会先创建一个value为0的hash，如果value不是数字类型，则会报错。该命令支持的数字范围是64位有符号整数。

``` bash
127.0.0.1:6379> keys * #演示使用，生产环境不要用
1) "mykey"
127.0.0.1:6379> HINCRBY myhash field1 1
(integer) 1
127.0.0.1:6379> HGET myhash field1
"1"
127.0.0.1:6379> HSET myhash fieldStr "follow"
(integer) 1
127.0.0.1:6379> HINCRBY myhash fieldStr 1
(error) ERR hash value is not an integer
127.0.0.1:6379> HGETALL myhash
1) "field1"
2) "1"
3) "fieldStr"
4) "follow"
127.0.0.1:6379> HINCRBY myhash field2 2
(integer) 2
127.0.0.1:6379> HGETALL myhash
1) "field1"
2) "1"
3) "fieldStr"
4) "follow"
5) "field2"
6) "2"
```



#### HINCRBYFLOAT

最早可用版本：2.6.0

时间复杂度：O(1)

用来对指定的key中指定的field进行浮点类型的加法，如果field不存在，则会先创建一个value为0的field。如果value或者increments不能解析为float类型，则会报错。通过下面的例子可以看到，浮点数的加法会存在一些偏差。

``` bash
127.0.0.1:6379> HINCRBYFLOAT myhash field3 0.3
"0.3"
127.0.0.1:6379> HINCRBYFLOAT myhash field3 1.0e3
"1000.29999999999999999"
127.0.0.1:6379> HINCRBYFLOAT myhash field3 -1.0e3
"0.29999999999999999"
127.0.0.1:6379> HINCRBYFLOAT myhash fieldStr 0.1
(error) ERR hash value is not a float
127.0.0.1:6379> HINCRBYFLOAT myhash field3 "haha"
(error) ERR value is not a valid float
```



#### HKEYS

最早可用版本：2.0.0

时间复杂度：O(N)，其中N为指定key中field的个数

HKEYS命令用于返回指定key中所包含的field列表，如果key不存在，则返回空列表。



#### HLEN

最早可用版本：2.0.0

时间复杂度：O(1)

返回指定的key所包含的field的个数。如果key不存在，则返回0。



#### HMGET

最早可用版本：2.0.0

时间复杂度：O(N)：N是请求的field的个数

返回指定key中指定的一个或多个field的值。如果field不存在，则返回nil，如果key不存在，同样会返回field数量的nil。因为不存在的key被作为空的hash处理。

``` bash
127.0.0.1:6379> HMGET myhash field1 field2 no-exist
1) "1"
2) "2"
3) (nil)
127.0.0.1:6379> HMGET no-exist field1 field2
1) (nil)
2) (nil)
```



#### HMSET

最早可用版本：2.0.0

时间复杂度：O(N)：N是需要设置的field的个数

为指定的key设置一个或多个field。如果field已经存在，则会被覆盖。如果指定的key不存在，则会创建一个新的hash。



#### HSCAN

最早可用版本：2.8.0

时间复杂度：每次请求的时间复杂度为O(1)，完成整个迭代的时间复杂度为O(N)

该命令与SCAN命令相似，可以参考我的另外一篇文章[Redis命令详解：Keys](https://jackeyzhe.github.io/2018/09/22/Redis%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3%EF%BC%9AKeys/#SCAN)中对SCAN用法的介绍，如果你想要有更深入了了解，可以看我的另外一篇文章[深入理解Redis的scan命令](https://jackeyzhe.github.io/2018/09/26/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Redis%E7%9A%84scan%E5%91%BD%E4%BB%A4/)。



#### HSET

最早可用版本：2.0.0

时间复杂度：O(1)

为指定的key中的field设置value，如果key不存在，则会创建一个新的hash，如果field已经存在，则会覆盖旧值。如果是新增的field，设置完成后会返回1，如果是更新已有的field，设置完成后会返回0。



#### HSETNX

最早可用版本：2.0.0

时间复杂度：O(1)

同样是为指定的key中的field设置value，与HSET命令不同的是，如果field已经存在，则不会有任何操作，直接返回0。



#### HSTRLEN

最早可用版本：3.2.0

时间复杂度：O(1)

返回指定key中field对应value的字符串长度，如果key或field不存在，返回0。



#### HVALS

最早可用版本：2.0.0

时间复杂度：O(N)，N为hash的size

返回指定key的hash的所有value。如果key不存在，则会返回空列表。