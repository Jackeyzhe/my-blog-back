---
title: 【译】antirez：Redis6将支持客户端缓存
date: 2019-09-07 22:11:38
tags: Redis
---

本文翻译自Redis作者antirez的一篇博客，原文地址是：http://antirez.com/news/130<!-- more -->

纽约Redis日已经结束了，我仍然与意大利时区同步，早上5点30起床，并立即走上了曼哈顿的街道，我很喜欢这里的风景，并且享受着成为这里的一部分。当时我正在考虑发布Redis 6的release版本，这是在未来一段时间最重要的事了。新版本的Redis协议（RESP3）推进得还很慢，如果没有一个好的理由，明智的人是不会更换工具的。但我为什么要坚持提升协议呢？有两个主要的原因，一是需要给客户端提供更加具有语义的回复，二是提供一个旧版本不能实现的新功能：客户端缓存。

时间倒回一年前，我到达圣安东尼奥的Redis Conf 2018。当时公司就有一个共识是客户端缓存是Redis在未来非常重要的事情。如果我们需要更快的存储和更快的缓存，我们就需要在客户端存储一部分信息。这是提供低延迟和大规模数据服务的很自然的想法。事实上，基本上每个大公司也都是这样做的，因为这是唯一的办法。然而Redis没有办法在这一过程中协助客户。一个巧合让Ben Malec想要在Redis Conf上做一些关于客户端缓存的演讲，他只使用Redis提供的工具和一些非常聪明的想法。

作者注：演讲地址是https://www.youtube.com/watch?v=kliQLwSikO4

Ben的演讲启发了我，为了实现Ben的设计，其中有两个关键点。第一个是使用Redis Cluster的“hash slot”的概念，把key分成了16k个组。采用这种方式使得客户端不需要追踪每一个key的位置，可以使用一个简单的元数据来定位key所在的group。Ben使用Pub/Sub模式来通知key的改变，所以他需要应用程序的一些帮助，然而这种模式是很固定的。要修改一个key？还需要发布失效消息。在客户端是否缓存了key呢？要记住缓存每个key和收到失效消息时的时间戳，记住每个slot的失效时间。当使用一个缓存的key时，先做一个懒清除，通过检查缓存key的时间戳是否早于slot收到失效信息的时间戳。这种情况下，这个key就是过时的数据，你可以再次访问服务器。

在看完演讲之后，我意识到这是一个在服务器内使用的好主意，为了让Redis能够为客户端做一部分工作，是客户端缓存更加简单高效，所以我回到家写下了我的设计文档：https://groups.google.com/d/msg/redis-db/xfcnYkbutDw/kTwCozpBBwAJ

但为了实现我的设计，我必须专注于修改Redis协议使它变得更加完善，所以我开始编写RESP3和Redis 6的其他特性（比如ACL）的规范和代码，客户端缓存是Redis许多迭代想法中的一种，有些想法因为时间不够放弃了。

当时我在纽约街头思考这个想法。后来和一些朋友去吃午饭喝咖啡。当我返回酒店房间后，距离第二天起飞还有一整晚的时间，所以我开始按照一年前写的提案来写Redis 6的客户端缓存的实现。

Redis服务器助理客户端缓存，最终叫做“tracking”（我也可能改主意），是一个由几个关键想法组成的非常简单功能。

key空间被分割到”caching slots“，但他们比Ben使用的hash slots要多得多。我们使用CRC64的24位输出，所以有超过1600万个不同的slot。为什么这么多呢？因为我认为你想要有一个1亿key的服务器。然而一个失效信息影响的key不应该多于客户端缓存中的key。Redis中失效表占用130M的内存：8字节的指针指向16M的条目。这对我来说是可以接受的，如果你想要使用新功能，你将充分利用你在客户端的所有内存，所以使用130MB在服务器端是好的，你可以获得更细粒度的失效。

客户端使用“opt in”方法开启这个功能，只需要一个简单的命令：

``` bash
CLIENT TRACKING on
```

服务器总是返回+OK，从这时起，每个命令都在命令表中被标记为“只读”，不再给调用者返回keys，并记住客户端请求的所有的key。保存这种信息时非常简单的，每个Redis客户端都有自己的唯一ID，所以如果ID是123的客户端发送了MGET命令，需要从slot 1，2和5获取key，那么失效表中我们就需要记录如下信息：

``` bash
1 -> [123]
2 -> [123]
5 -> [123]
```

接着，ID为444的客户端也需要到slot5请求key了，那么表信息将变成：

``` bash
5 -> [123,444]
```

现在其他客户端修改了slot 5中的某个key，Redis将会检查失效表，发现客户端123和444都缓存了这个slot上的key。我们将会给这些客户端发送失效信息，然后会记录下slot最后的失效时间戳，并在以后懒检查缓存对象的时间戳，并对照后判断是否失效。此外，客户端可以回收表中缓存的指定slot的对象。这种具有24位hash函数的方法不是问题，因为我们即使缓存几千万的key，也不会有很长的列表。发送了失效信息后，我们就可以删除失效表中的项，这样直到这些客户端不再读这些slot的key，我们就不再向他们发送失效消息。

需要注意的是，客户端不必强制使用24位hash函数。也可能使用20位，然后移动Redis发送的失效消息的slot。不确定是否有很多很好的理由这样做，但是内存受限时，这可能是一种想法。

如果你密切关注我说的话，你会开始考虑同一连接既会接收到正常的客户端回复，又会接收失效消息。这可以通过RESP3实现，因为失效作为“推送”消息类型发送。如果客户端是一个阻塞类型的，并且不是事件驱动类型的客户端，就会变得比较复杂：

应用程序需要一些方法来不时读取新数据，这看起来既复杂又脆弱。在这种情况下，为了接收失效消息，使用另一个应用程序线程和不同的客户端可能会更好。所以你可以使用以下命令来允许这样的操作：

``` bash
CLIENT TRACKING on REDIRECT 1234
```

基本上我们可以说我们使用当前连接获得的所有key，并希望失效消息发送到客户端1234。在连接池的情况下多个客户端可能会要求将失效消息重定向到单个客户端。你需要做的就是创建特殊连接以接收失效消息，调用CLIENT ID以了解此客户端连接哪个ID，然后启用跟踪。

现在只剩下一个问题了：如果我们失去了失效连接怎么办？我们可能因为不能接收到失效消息而陷入麻烦。通常，应用会检测连接，尝试重连，并清除缓存。为了确保失效连接处于连接状态，不时地向服务器发送ping请求可能是一个更好的主意。然而，为了降低过期数据的风险，Redis也将开始通知客户端将失效消息重定向到其他客户端，只要使用特殊的推送消息：下一个请求就会使客户端知道连接已经断开。

我刚才描述的已经合并到Redis的unstable分支。可能不是最终的处理方法，但是在第一个Redis 6发布版本之前还有几个月的时间，我们还有时间修改所有的事情：可以告诉我你的反馈。我也会再寻找其他RESP2可行的方法。这只有在重定向开启时才有效，并且客户端要进入Pub/Sub模式监听消息。通过这种方式，完全可以复用旧客户端。

我希望这足以刺激你的胃口：如果我们在Redis中运行的很好，然后记录下来，让客户端作者知道该如何支持，数据可能比以往更接近应用程序，甚至在小型团队运行的应用程序中，到目前为止还没有尝试客户端缓存。对于正在准备做的大型团队和非常大的应用程序，降低实现成本和复杂性。