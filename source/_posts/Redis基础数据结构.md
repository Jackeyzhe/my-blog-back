---
title: Redis基础数据结构
date: 2018-09-17 23:04:19
tags: Redis
---

Redis是一款完全免费的，高性能的key-value数据库，目前被大多数公司用来做缓存。Redis作为一个内存数据库，它的读写速度非常快：读速度可以达到110000次/s，写的速度是81000次/s 。相比于其他key-value数据库，Redis的另一大特性就是支持多种数据类型。今天我们来一起聊一聊Redis的5种基础数据类型。<!-- more -->

#### 安装Redis

在学习之前，我们要先自己安装一个Redis环境用来自己动手操作，感受一下。

##### Windows

**下载地址：**https://github.com/MSOpenTech/redis/releases

![Redis_window_pack](https://res.cloudinary.com/dxydgihag/image/upload/v1537284281/Blog/Redis/Redis_windows_path.png)

Windows用户可以在这个地址下载相应版本的压缩包，在C盘进行解压，解压后，将目录重命名为redis。在cmd中进入该目录，然后运行redis-server.exe redis.windows.conf。另外，也可以把目录加到环境变量中，这样就不需要再cd进入这个目录了。

Redis的server安装好后，再打开一个新的cmd， 运行redis-cli.exe -h 127.0.0.1 -p 6379，就可以开始进行操作了。其中-h参数表示host，-p参数表示port，可以省略，默认是6379。



##### Linux

```bash
$ wget http://download.redis.io/releases/redis-4.0.11.tar.gz
$ tar xzf redis-4.0.11.tar.gz
$ cd redis-4.0.11
$ make
```

执行以上命令下载并安装Redis，接着进入src目录，运行redis-server。再执行 

``` bash
 $ ./redis-cli 
```

命令，就可以开始操作了。



##### Ubuntu

Ubuntu可以直接使用apt-get安装

``` bash
$sudo apt-get update
$sudo apt-get install redis-server
```

启动方法这里不再赘述。



##### Mac

Mac用户可以使用homebrew安装Redis

``` bash
brew install redis
```



##### 其他方法

除了上述方法以外，我们还可以从GitHub下载源码，对源码进行编译。URL是git@github.com:antirez/redis.git。也可以从[官网](https://redis.io/download)下载Docker，通过运行Docker来操作。



#### 基础数据类型

Redis支持5种基础数据类型，下面我们来一一介绍，由于我本身是Java程序员，因此会将这些数据类型与Java中的数据类型进行类比。当然，你也可以拿自己熟悉的语言来理解。



##### String

String是最基本的，也是最常用的类型。它是二进制安全的，也就是说，我们可以将对象序列化成json字符串作为value值存入Redis。在分配内存时，Redis会为一个字符串分配一些冗余的空间，以避免因字符串的值改变而出现频繁的内存分配操作。当字符串长度小于1M时，每次扩容都会加倍现有空间，当长度大于1M时，每次扩容，增加1M，Redis字符串的最大长度是512M。



##### Hash

Hash是键值对集合，相当于Java中的HashMap，实际结构也和HashMap一样，是数组+链表的结构。所不同的是扩容的方式不同，HashMap是进行一次rehash，而Redis为了不阻塞服务，会创建一个新的数组，在查询时会同时查询两个Hash，然后在逐渐将旧的Hash内容转移到新的中去。一个Hash最大可以存储2<sup>32</sup>-1个键值对。



##### List

List相当于Java中的LinkedList，它的插入和删除操作的时间复杂度为O(1)，而查询操作的时间复杂度为O(n)。我们可以利用List的rpush、rpop、lpush和lpop命令来构建队列或者栈。列表最多可以存储2<sup>32</sup>-1个元素。



##### Set

Set是String类型的无序集合，并且元素唯一，相当于Java中的HashSet，它的插入、删除、查询操作的时间复杂度都是O(1)。其最大元素数也是2<sup>32</sup>-1个。



##### zset

zset可以看做是Java中SortedSet和HashMap的结合，一方面它不允许元素重复，另一方面，它通过score为每个元素进行排序。



#### 两个规则

对于以上5种数据结构，有两个通用的规则：

1. 如果不存在，就先创建，再进行操作
2. 如果元素为空，就会释放内存



#### 过期时间

我们可以对上面所有的类型设置过期时间，如果时间到了，Redis 会自动删除相应的对象。



#### 小结

本文简单介绍了Redis的安装方法和Redis的5中基本数据结构。主要目的是帮助没有基础的同学快速入门，对于已经了解Redis的同学也是知识的巩固，想要了解更多关于Redis的知识，可以持续关注我，后面还有更精彩的内容分享给大家。

