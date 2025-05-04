---
title: Redis命令详解：Pub/Sub
date: 2019-08-28 00:37:57
tags: Redis命令
---

Redis发布订阅模式相关命令。<!-- more -->

#### PSUBSCRIBE

最早可用版本：2.0.0

时间复杂度：O(N)，N是已订阅的客户端数。

订阅给定规则的客户端，支持的形式包括：

- h?llo 订阅hello,hallo和hxllo等
- h*llo 订阅hllo和heeeello等
- h[ae] 订阅hello和hallo，但不订阅hillo

如果要逐字匹配，要使用\来转义特殊字符。

#### PUBLISH

最早可用版本：2.0.0

时间复杂度：O(N+M)，N是已订阅的客户端数，M是订阅总数

发布消息到指定频道。

#### PUBSUB

最早可用版本：2.8.0

时间复杂度：O(N)，N是活跃的频道数

该命令用于检查Pub/Sub子系统的状态。

``` bash
PUBSUB CHANNELS [pattern]
```

列出当前活跃的频道（至少有一个订阅者）。不过不指定pattern，则列出全部频道。

``` bash
PUBSUB NUMSUB [channel-1 ... channel-N]
```

返回指定频道的订阅者。

``` bash
PUBSUB NUMPAT
```

返回指定模式的订阅数（使用PSUBSCRIBE命令执行）

#### PUNSUBSCRIBE

最早可用版本：2.0.0

时间复杂度：O(N+M)，N是匹配规则的客户端已经订阅的数量，M是系统中匹配规则的订阅总数 

用法：PUNSUBSCRIBE [pattern [pattern ...]]

退订所有匹配规则的频道，如果没有指定规则，则退订所有的频道。

#### SUBSCRIBE

最早可用版本：2.0.0

时间复杂度：O(N)，N是订阅频道的数量

给客户端订阅指定的频道。

#### UNSUBSCRIBE

最早可用版本：2.0.0

时间复杂度：O(N)，N是订阅频道的数量

给客户端退订指定的频道。如果不指定频道，则退订全部。