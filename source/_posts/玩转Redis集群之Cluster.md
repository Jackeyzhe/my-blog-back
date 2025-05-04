---
title: 玩转Redis集群之Cluster
date: 2018-11-27 23:17:43
tags: Redis
---

前面我们介绍了国人自己开发的Redis集群方案——Codis，Codis友好的管理界面以及强大的自动平衡槽位的功能深受广大开发者的喜爱。今天我们一起来聊一聊Redis作者自己提供的集群方案——Cluster。希望读完这篇文章，你能够充分了解Codis和Cluster各自的优缺点，面对不同的应用场景可以从容的做出选择。<!-- more -->

Redis Cluster是去中心化的，这点与Codis有着本质的不同，Redis Cluster划分了16384个slots，每个节点负责其中的一部分数据。slot的信息存储在每个节点中，节点会将slot信息持久化到配置文件中，因此需要保证配置文件是可写的。当客户端连接时，会获得一份slot的信息。这样当客户端需要访问某个key时，就可以直接根据缓存在本地的slot信息来定位节点。这样就会存在客户端缓存的slot信息和服务器的slot信息不一致的问题，这个问题具体怎么解决呢？这里先卖个关子，后面会做解释。

#### 特性

首先我们来看下官方对Redis Cluster的介绍。

- High performance and linear scalability up to 1000 nodes. There are no proxies, asynchronous replication is used, and no merge operations are performed on values.
- Acceptable degree of write safety: the system tries (in a best-effort way) to retain all the writes originating from clients connected with the majority of the master nodes. Usually there are small windows where acknowledged writes can be lost. Windows to lose acknowledged writes are larger when clients are in a minority partition.
- Availability: Redis Cluster is able to survive partitions where the majority of the master nodes are reachable and there is at least one reachable slave for every master node that is no longer reachable. Moreover using *replicas migration*, masters no longer replicated by any slave will receive one from a master which is covered by multiple slaves.

是不是不(kan)想(bu)看(dong)？没关系，我来给你掰开了揉碎了解释一下。

##### 写安全

Redis Cluster使用异步的主从同步方式，只能保证最终一致性。所以会引起一些写入数据丢失的问题，在继续阅读之前，可以先自己思考一下在什么情况下写入的数据会丢失。

先来看一种比较常见的写丢失的情况：

client向一个master发送一个写请求，master写成功并通知client。在同步到slave之前，这个master挂了，它的slave代替它成为了新的master。这时前面写入的数据就丢失了。

此外，还有一种情况。

master节点与大多数节点无法通信，一段时间后，这个master被认为已经下线，并且被它的slave顶替，又过了一段时间，原来的master节点重写恢复了连接。这时如果一个client存有过期的路由表，它就会把写请求发送的这个旧的master节点（已经变成slave了）上，从而导致写数据丢失。

不过，这种情况一般不会发生，因为当一个master失去连接足够长时间而被认为已经下线时，就会开始拒绝写请求。当它恢复之后，仍然会有一小段时间是拒绝写请求的，这段时间是为了让其他节点更新自己的路由表中的配置信息。

为了尽可能保证写安全性，Redis Cluster在发生分区时，会尽量使客户端连接到多数节点的那一部分，因为如果连接到少数部分，当master被替换时，会因为多数master不可达而拒绝所有的写请求，这样损失的数据要增大很多。

Redis Cluster维护了一个NODE_TIMEOUT变量，如果上述情况中，master在NODE_TIMEOUT时间内恢复连接，就不会有数据丢失。

##### 可用性

如果集群的大部分master可达，并且每个不可达的master至少有一个slave，在NODE_TIMEOUT时间后，就会开始进行故障转移（一般1到2秒），故障转移完成后的集群仍然可用。

如果集群中得N个master节点都有1个slave，当有一个节点挂掉时，集群一定是可用的，如果有2个节点挂掉，那么就会有1/(N*2-1)的概率导致集群不可用。

Redis Cluster为了提高可用性，新增了一个新的feature，叫做replicas migration（副本迁移，ps：我自己翻译的），这个feature其实就是在每次故障之后，重新布局集群的slave，给没有slave的master配备上slave，以此来更好的应对下次故障。

##### 性能

Redis Cluster不提供代理，而是让client直接重定向到正确的节点。

client中会保存一份集群状态的副本，一般情况下就会直接连接到正确的节点。

由于Redis Cluster是异步备份的，所以节点不需要等待其他节点确认写成功就可以直接返回，除非显式的使用了WAIT命令。

对于操作多个key的命令，所操作的key必须是在同一节点上的，因为数据是不会移动的。（除非是resharding）

Redis Cluster设计的主要目标是提高性能和扩展性，只提供弱的数据安全性和可用性（但是要合理）。

#### Key分配模型

Redis Cluster共划分为16384个槽位。这也意味着一个集群最多可以有16384个master，不过官方建议master的最大数量是1000个。

如果Cluster不处于重新配置过程，那么就会达到一种稳定状态。在稳定状态下，一个槽位只由一个master提供服务，不过一个master节点会有一个或多个slave，这些slave可以提供缓解master的读请求的压力。

Redis Cluster会对key使用[CRC16](https://www.cnblogs.com/94cool/p/3559585.html)算法进行hash，然后对16384取模来确定key所属的槽位（hash tag会打破这种规则）。

#### Keys hash tags

标签是破坏上述计算规则的实现，Hash tag是一种保证多个键被分配到同一个槽位的方法。

hash tag的计算规则是：取一对大括号{}之间的字符进行计算，如果key存在多对大括号，那么就取第一个左括号和第一个右括号之间的字符。如果大括号之前没有字符，则会对整个字符串进行计算。

说了这个多，可能你还是一头雾水。别急，我们来吃几个栗子。

1. {Jackeyzhe}.following和{Jackeyzhe}.follower这两个key都是计算Jackey的hash值
2. foo{{bar}}这个key就会对{bar进行hash计算
3. follow{}{Jackey}会对整个字符串进行计算

#### 重定向

前面聊性能的时候我们提到过，Redis Cluster为了提高性能，不会提供代理，而是使用重定向的方式让client连接到正确的节点。下面我们来详细说明一下Redis Cluster是如何进行重定向的。

##### MOVED重定向

Redis客户端可以向集群的任意一个节点发送查询请求，节点接收到请求后会对其进行解析，如果是操作单个key的命令或者是包含多个在相同槽位key的命令，那么该节点就会去查找这个key是属于哪个槽位的。

如果key所属的槽位由该节点提供服务，那么就直接返回结果。否则就会返回一个MOVED错误：

``` bash
GET x
-MOVED 3999 127.0.0.1:6381
```

这个错误包括了对应的key属于哪个槽位（3999）以及该槽位所在的节点的IP地址和端口号。client收到这个错误信息后，就将这些信息存储起来以便可以更准确的找到正确的节点。

当客户端收到MOVED错误后，可以使用CLUSTER NODES或CLUSTER SLOTS命令来更新整个集群的信息，因为当重定向发生时，很少会是单个槽位的变更，一般都会是多个槽位一起更新。因此，在收到MOVED错误时，客户端应该尽早更新集群的分布信息。当集群达到稳定状态时，客户端保存的槽位和节点的对应信息都是正确的，cluster的性能也会达到非常高效的状态。

除了MOVED重定向之外，一个完整的集群还应该支持ASK重定向。

##### ASK重定向

对于Redis Cluster来讲，MOVED重定向意味着请求的slot永远由另一个node提供服务，而ASK重定向仅代表下一个请求需要发送到指定的节点。在Redis Cluster迁移的时候会用到ASK重定向，那Redis Cluster迁移的过程究竟是怎样的呢？

Redis Cluster的迁移是以槽位单位的，迁移过程总共分3步（类似于把大象装进冰箱），我们来举个栗子，看一下一个槽位从节点A迁移到节点B需要经过哪些步骤：

1. 首先打开冰箱门，也就是从A节点获得槽位所有的key列表，再挨个key进行迁移，在这之前，A节点的该槽位被设置为migrating状态，B节点被设置为importing的槽位（都是用CLUSTER SETSLOT命令）。
2. 第二步，就是要把大象装进去了，对于每个key来说，就是在A节点用dump命令对其进行序列化，再通过客户端在B节点执行restore命令，反序列化到B节点。
3. 第三步呢，就需要把冰箱门关上，也就是把对应的key从A节点删除。

有同学会问了，说好的用到ASK重定向呢？上面我们所描述的只是迁移的过程，在迁移过程中，Redis还是要对外提供服务的。试想一下，如果在迁移过程中，我向A节点请求查询x的值，A说：我这没有啊，我也不知道是传到B那去了还是我一直就没有存，你还是先问问B吧。然后返回给我们一个-ASK targetNodeAddr的错误，让我们去问B。而这时如果我们直接去问B，B肯定会直接说：这个不归我管，你得去问A。（-MOVED重定向）。因为这时候迁移还没有完成，所以B也没说错，这时候x真的不归它管。但是我们不能让它俩来回踢皮球啊，所以在问B之前，我们先给B发一个asking指令，告诉B：下面我问你一个key的值，你得当成是自己的key来处理，不能说不知道。这样如果x已经迁移到B，就会直接返回结果，如果B也查不到x的下落，说明x不存在。

#### 容错

了解了Redis Cluster的重定向操作之后，我们再来聊一聊Redis Cluster的容错机制，Redis Cluster和大多数集群一样，是通过心跳来判断一个节点是否存活的。

##### 心跳和gossip消息

集群中的节点会不停的互相交换ping pong包，ping pong包具有相同的结构，只是类型不同，ping pong包合在一起叫做心跳包。

通常节点会发送ping包并接收接收者返回的pong包，不过这也不是绝对，节点也有可能只发送pong包，而不需要让接收者发送返回包，这种操作通常用于广播一个新的配置信息。

节点会每个几秒钟就发送一定数量的ping包。如果一个节点超过二分之一NODE_TIME时间没有收到来自某个节点ping或pong包，那么就会在NODE_TIMEOUT之前像该节点发送ping包，在NODE_TIMEOUT之前，节点会尝试TCP重连，避免由于TCP连接问题而误以为节点不可达。

##### 心跳包内容

前面我们说了，ping和pong包的结构是相同的，下面就来具体看一下包的内容。

ping和pong包的内容可以分为header和gossip消息两部分，其中header包含以下信息：

- NODE ID是一个160bit的伪随机字符串，它是节点在集群中的唯一标识
- currentEpoch和configEpoch字段
- node flag，标识节点是master还是slave，另外还有一些其他的标识位
- 节点提供服务的hash slot的bitmap
- 发送者的TCP端口
- 发送者认为的集群状态（down or ok）
- 如果是slave，则包含master的NODE ID

gossip包含了该节点认为的其他节点的状态，不过不是集群的全部节点。具体有以下信息：

- NODE ID
- 节点的IP和端口
- NODE flags

gossip消息在错误检测和节点发现中起着重要的作用。

#### 错误检测

错误检测用于识别集群中的不可达节点是否已下线，如果一个master下线，会将它的slave提升为master。如果无法提升，则集群会处于错误状态。在gossip消息中，NODE flags的值包括两种PFAIL和FAIL。

##### PFAIL flag

如果一个节点发现另外一个节点不可达的时间超过NODE_TIMEOUT ，则会将这个节点标记为PFAIL，也就是Possible failure（可能下线）。节点不可达是说一个节点发送了ping包，但是等待了超过NODE_TIMEOUT时间仍然没有收到回应。这也就意味着，NODE_TIMEOUT必须大于一个网络包来回的时间。

##### FAIL flag

PFAIL标志只是一个节点本地的信息，为了使slave提升为master，需要将PFAIL升级为FAIL。PFAIL升级为FAIL需要满足一些条件：

- A节点将B节点标记为PFAIL
- A节点通过gossip消息收集其他大部分master节点标识的B节点的状态
- 大部分master节点在NODE_TIMEOUT * FAIL_REPORT_VALIDITY_MULT时间段内，标识B节点为PFAIL或FAIL

如果满足以上条件，A节点会将B节点标识为FAIL并且向所有节点发送B节点FAIL的消息。收到消息的节点也都会将B标为FAIL。

FAIL状态是单向的，只能从PFAIL升级为FAIL，而不能从FAIL降为PFAIL。不过存在一些清除FAIL状态的情况：

- 节点重新可达，并且是slave节点
- 节点重新可达，并且是master节点，但是不提供任何slot服务
- 节点重新可达，并且是master节点，但是长时间没有slave被提升为master来顶替它

PFAIL提升到FAIL使用的是一种弱协议：

- 节点收集的状态不在同一时间点，我们会丢弃时间较早的报告信息，但是也只能保证节点的状态在一段时间内大部分master达成了一致
- 检测到一个FAIL后，需要通知所有节点，但是没有办法保证每个节点都能成功收到消息

由于是弱协议，Redis Cluster只要求所有节点对某个节点的状态最终保持一致。如果大部分master认为某个节点FAIL，那么最终所有节点都会将其标为FAIL。而如果只有一小部分master节点认为某个节点FAIL，slave并不会被提升为master，因此，FAIL状态将会被清除。

#### 搭建

原理说了这么多，我们一定要来亲自动手搭建一个Redis Cluster，下面演示一个在一台机器上模拟搭建3主3从的Redis Cluster。当然，如果你想了解更多Redis Cluster的其他原理，可以查看[官网](https://redis.io/topics/cluster-spec)的介绍。

##### Redis环境

首先要搭建起我们需要的Redis环境，这里启动6个Redis实例，端口号分别是6379、6380、6479、6480、6579、6580

拷贝6份Redis配置文件并进行如下修改（以6379为例，端口号和配置文件根据需要修改）：

``` bash
port 6379
cluster-enabled yes
cluster-config-file nodes6379.conf
appendonly yes
```

配置文件的名称也需要修改，修改完成后，分别启动6个实例（图片中有一个端口号改错了……）。

![Redis instances](https://res.cloudinary.com/dxydgihag/image/upload/v1544443793/Blog/Redis/run_redis_instance.png)

##### 创建Redis Cluster

实例启动完成后，就可以创建Redis Cluster了，如果Redis的版本是3.x或4.x，需要使用一个叫做redis-trib的工具，而对于Redis5.0之后的版本，Redis Cluster的命令已经集成到了redis-cli中了。这里我用的是Redis5，所以没有再单独安装redis-trib工具。

接下来执行命令

``` bash
redis-cli --cluster create 127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6479 127.0.0.1:6480 127.0.0.1:6579 127.0.0.1:6580 --cluster-replicas 1
```

当你看到输出了

``` bash
[OK] All 16384 slots covered
```

就表示Redis Cluster已经创建成功了。

##### 查看节点信息

此时我们使用cluster nodes 命令就可查看Redis Cluster的节点信息了。

![cluster nodes](https://res.cloudinary.com/dxydgihag/image/upload/v1544443754/Blog/Redis/clusters_nodes.png)

可以看到，6379、6380和6479三个节点被配置为master节点。

##### reshard

接下来我们再来尝试一下reshard操作

![reshard_start](https://res.cloudinary.com/dxydgihag/image/upload/v1544443793/Blog/Redis/reshard_start.png)

如图，输入命令

``` bash
redis-cli --cluster reshard 127.0.0.1:6380
```

Redis Cluster会问你要移动多少个槽位，这里我们移动1000个，接着会询问你要移动到哪个节点，这里我们输入6479的NODE ID

![reshard_end](https://res.cloudinary.com/dxydgihag/image/upload/v1544443793/Blog/Redis/reshard_end.png)

reshard完成后，可以输入命令查看节点的情况

``` bash
redis-cli --cluster check 127.0.0.1:6480
```

可以看到6479节点已经多了1000个槽位了，分别是0-498和5461-5961。

##### 新增master节点

![add_node](https://res.cloudinary.com/dxydgihag/image/upload/v1544443769/Blog/Redis/add_node.png)我们可以使用add-node命令为Redis Cluster新增master节点，可以看到我们增加的是6679节点，新增成功后，并不会为任何slot提供服务。

##### 新增slave节点

![add_slave](https://res.cloudinary.com/dxydgihag/image/upload/v1544443767/Blog/Redis/add_node_slave.png)

我们也可以用add-node命令新增slave节点，只不过需要加上--cluster-slave参数，并且使用--cluster-master-id指明新增的slave属于哪个master。

#### 总结

最后来总结一下，我们介绍了

Redis Cluster的特性：写安全、可用性、性能

Key分配模型：使用CRC16算法，如果需要分配到相同的slot，可以使用tag

两种重定向：MOVED和ASK

容错机制：PFAIL和FAIL两种状态

最后又动手搭建了一个实验的Redis Cluster。