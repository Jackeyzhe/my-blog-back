---
title: Redis总结篇
date: 2019-09-07 09:31:53
tags: Redis
---

Redis的文章已经写了很长时间了，在这期间，也依靠对Redis的熟悉在面试过程中获得了一些加分。在新的工作中也面临了新的挑战，因此决定对Redis的文章暂时告一段落，这里也对之前的学习进行一下总结。<!-- more -->

#### 第一步

首先，我们要了解什么是Redis，并尝试安装Redis，以方便后面进行一些试验。然后就是掌握最基础的数据结构，这在《[Redis基础数据结构](https://jackeyzhe.github.io/2018/09/17/Redis%E5%9F%BA%E7%A1%80%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)》一文中都有介绍。

#### 命令

在有了基础之后，就可以开始尝试进行一些实际操作。对Redis命令的了解是少不了的。各个命令按照功能可以分为以下类别：

- [Connection](https://jackeyzhe.github.io/2018/09/19/Redis命令详解：Connection/)
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

在有了这些基础后，我们知道了生产环境中是禁止使用keys命令的，通常使用scan命令来查询/遍历key。所以我们在《[深入理解Redis的scan命令](https://jackeyzhe.github.io/2018/09/26/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Redis%E7%9A%84scan%E5%91%BD%E4%BB%A4/)》一文中对SCAN命令有了更详细的介绍。

#### 集群

当然，只知道这些还不够，在实际工作中，Redis通常以集群的方式部署，所以我们又介绍了部署Redis集群的三种方式。其中包括：

- 哨兵模式：《[玩转Redis集群之Sentinel](https://jackeyzhe.github.io/2018/11/04/玩转Redis集群之Sentinel/)》
- Codis代理：《[玩转Redis集群之Codis](https://jackeyzhe.github.io/2018/11/14/%E7%8E%A9%E8%BD%ACRedis%E9%9B%86%E7%BE%A4%E4%B9%8BCodis/)》
- 官方集群Cluster：《[玩转Redis集群之Cluster](https://jackeyzhe.github.io/2018/11/27/%E7%8E%A9%E8%BD%ACRedis%E9%9B%86%E7%BE%A4%E4%B9%8BCluster/)》

开发Codis的团队现在还做了分布式MySQL——[TiDB](https://github.com/pingcap/tidb)，感兴趣的同学可以了解一下。

#### 源码

学到这里，你已经学会了“怎么用”，但是作为一名优秀的程序员，一定不能就此止步，还应该知道你用的东西究竟是怎么做出来的。因此，我们一起走近了源码，对Redis命令执行过程，以及一些底层存储方式做了更加深入的了解。

[走近源码：Redis的启动过程](https://jackeyzhe.github.io/2019/01/04/%E8%B5%B0%E8%BF%91%E6%BA%90%E7%A0%81%EF%BC%9ARedis%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/)

[走近源码：Redis如何执行命令](https://jackeyzhe.github.io/2019/01/05/%E8%B5%B0%E8%BF%91%E6%BA%90%E7%A0%81%EF%BC%9ARedis%E5%A6%82%E4%BD%95%E6%89%A7%E8%A1%8C%E5%91%BD%E4%BB%A4/)

[走近源码：Redis命令执行过程（客户端）](https://jackeyzhe.github.io/2019/01/12/%E8%B5%B0%E8%BF%91%E6%BA%90%E7%A0%81%EF%BC%9ARedis%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B%EF%BC%88%E5%AE%A2%E6%88%B7%E7%AB%AF%EF%BC%89/)

[走近源码：神奇的HyperLogLog](https://jackeyzhe.github.io/2019/02/26/%E8%B5%B0%E8%BF%91%E6%BA%90%E7%A0%81%EF%BC%9A%E7%A5%9E%E5%A5%87%E7%9A%84HyperLogLog/)

[走近源码：压缩列表是怎样炼成的](https://jackeyzhe.github.io/2019/03/23/%E8%B5%B0%E8%BF%91%E6%BA%90%E7%A0%81%EF%BC%9A%E5%8E%8B%E7%BC%A9%E5%88%97%E8%A1%A8%E6%98%AF%E6%80%8E%E6%A0%B7%E7%82%BC%E6%88%90%E7%9A%84/)

[走近源码：Redis跳跃列表究竟怎么跳](https://jackeyzhe.github.io/2019/04/18/%E8%B5%B0%E8%BF%91%E6%BA%90%E7%A0%81%EF%BC%9ARedis%E8%B7%B3%E8%B7%83%E5%88%97%E8%A1%A8%E7%A9%B6%E7%AB%9F%E6%80%8E%E4%B9%88%E8%B7%B3/)

#### 其他

最后，我们还了解了一些其他的技术，包括管道、Lua以及Redis的通信协议。

[速度不够，管道来凑——Redis管道技术](https://jackeyzhe.github.io/2019/04/27/%E9%80%9F%E5%BA%A6%E4%B8%8D%E5%A4%9F%EF%BC%8C%E7%AE%A1%E9%81%93%E6%9D%A5%E5%87%91%E2%80%94%E2%80%94Redis%E7%AE%A1%E9%81%93%E6%8A%80%E6%9C%AF/)

[Redis Lua脚本小学教程](https://jackeyzhe.github.io/2019/05/13/Redis-Lua%E8%84%9A%E6%9C%AC%E5%B0%8F%E5%AD%A6%E6%95%99%E7%A8%8B/)

[Redis Lua脚本中学教程（上）](https://jackeyzhe.github.io/2019/06/10/Redis-Lua%E8%84%9A%E6%9C%AC%E4%B8%AD%E5%AD%A6%E6%95%99%E7%A8%8B%EF%BC%88%E4%B8%8A%EF%BC%89/)

[Redis Lua脚本中学教程（下）](https://jackeyzhe.github.io/2019/06/16/Redis-Lua%E8%84%9A%E6%9C%AC%E4%B8%AD%E5%AD%A6%E6%95%99%E7%A8%8B%EF%BC%88%E4%B8%8B%EF%BC%89/)

[Redis Lua脚本大学教程](https://jackeyzhe.github.io/2019/06/17/Redis-Lua%E8%84%9A%E6%9C%AC%E5%A4%A7%E5%AD%A6%E6%95%99%E7%A8%8B/)

[浅谈Redis通信协议](https://jackeyzhe.github.io/2019/06/23/%E6%B5%85%E8%B0%88Redis%E9%80%9A%E4%BF%A1%E5%8D%8F%E8%AE%AE/)

#### 未来

Redis的相关知识远远不止这些，所以我还要和大家一起继续学习。这里推荐一些学习资料：

- [Redis官网](https://redis.io/)
- [作者antirez的博客](http://antirez.com/latest/0)
- [Redis设计与实现](https://book.douban.com/subject/25900156/)
- 老钱的Redis小册

![老钱小册](https://res.cloudinary.com/dxydgihag/image/upload/v1567827703/Blog/Redis/%E8%80%81%E9%92%B1Redis.png)

