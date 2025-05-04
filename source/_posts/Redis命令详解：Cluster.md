---
title: Redis命令详解：Cluster
date: 2019-08-28 01:15:56
tags: Redis命令
---

前文中我们介绍过了Redis的三种集群方案，没有了解过的同学可以自行前往。今天要介绍的Redis的亲儿子Cluster相关的命令。<!-- more -->

[玩转Redis集群之Sentinel](https://jackeyzhe.github.io/2018/11/04/%E7%8E%A9%E8%BD%ACRedis%E9%9B%86%E7%BE%A4%E4%B9%8BSentinel/)

[玩转Redis集群之Codis](https://jackeyzhe.github.io/2018/11/14/%E7%8E%A9%E8%BD%ACRedis%E9%9B%86%E7%BE%A4%E4%B9%8BCodis/)

[玩转Redis集群之Cluster](https://jackeyzhe.github.io/2018/11/27/%E7%8E%A9%E8%BD%ACRedis%E9%9B%86%E7%BE%A4%E4%B9%8BCluster/)

#### CLUSTER ADDSLOTS

最早可用版本：3.0.0

时间复杂度：O(N)，N是参数中hash的slot总数

这个命令是用来将指定的slot分配给接收命令的机器。如果执行成功，该机器就拥有这些slot，并会在集群中进行广播。

需要注意的是：

1. 该命令只有在当所有指定的slot在接收命令的节点上没有被分配时生效。节点将拒绝接纳已经分配到其他节点的slot。
2. 如果相同的slot被多次指定，命令就会执行失败。
3. 如果一个slot作为参数被设置为importing，一旦节点向自己分配该slot，这个状态将会被清除。

这个命令的应用场景有两种：

1. 创建新的集群时，ADDSLOTS用于主节点初始化分配可用的hash slots
2. 修复有未分配slots的坏集群

如果一个节点为自己分配了一个slot集合，它会将这个信息在心跳包的header里传播出去。然而其他节点只有在他们的slot没有被其他节点绑定或者没有作为新节点传播时才会接收这个信息。

这意味着这个命令应该仅通过redis集群应用管理客户端，例如redis-trib。如果这个命令使用了错误的上下文会导致集群处于错误的状态或者导致数据丢失，因此这个命令需要谨慎使用。

#### CLUSTER COUNT-FAILURE-REPORTS

最早可用版本：3.0.0

时间复杂度：O(N)，N是故障报告的数量

这个命令返回指定节点的故障报告。故障报告是Redis Cluster用来将节点从PFAIL状态转换到FAIL状态的方式。

更多的细节：

- 一个节点会用PFAIL标记一个不可达时间超过超时时间，这个超时时间是Redis Cluster配置中的基本选项
- 处于PFAIL状态的节点会将状态信息提供在心跳包的gossip部分。
- 每当一个节点处理来自其他节点的gossip信息时，该节点会建立故障报告，并且会记住发送消息包的节点说的其他节点是在PFAIL状态的消息。
- 每个故障报告的生存时间是节点超时时间的两倍
- 如果在一段时间一个节点被另一个节点标记为PFAIL状态，并且在同一时间收到大多数主节点关于该节点的故障报告，那么该节点的故障状态会从PFAIL变成FAIL，并且广播这个信息，让所有可达的节点将这个节点标记为FAIL。

该节点返回当前节点的故障报告数，该计数值不包括当前节点。

#### CLUSTER COUNTKEYSINSLOT

最早可用版本：3.0.0

时间复杂度：O(1)

这个命令返回指定Redis集群的slot的key的数量。该命令只查询连接节点的本地数据集，如果指定的slot被分配在别的节点上，就会返回0。

#### CLUSTER DELSLOTS

最早可用版本：3.0.0

时间复杂度：O(N)，N是slot参数的数量

在Redis Cluster中，每个节点都会知道哪些主节点正在负责哪些slot。

DELSLOTS命令使一个特定的节点忘记主节点负责的hash slot。在这之后，这些hash slot就被认为是未绑定状态的。需要注意的是：

1. 命令只在参数指定的hash slot和某些节点绑定时有效
2. 如果同一个hash slot被指定多次，该命令会失效
3. 节点可能因为没有覆盖全部slot而变成下线状态

#### CLUSTER FAILOVER

最早可用版本：3.0.0

时间复杂度：O(1)

用法：CLUSTER FAILOVER [FORCE|TAKEOVER]

该命令只能再集群slave节点执行，让slave节点进行一次人工的故障切换。

人工故障切换时一种常规操作，而不是真的出了故障。当我们希望当前的master和它的slave进行一次安全的主备切换时，流程如下：

1. 当前slave节点告知其master停止处理来自客户端的请求。
2. master节点将当前replication offset回复给slave
3. 该slave节点在未应用至replication offset之前不做任何操作，以保证master传来的数据均被处理
4. 该slave节点进行故障转移，从集群中大多数master获取一个新的配置，并广播自己的最新配置
5. 旧的master更新配置，解除客户端阻塞，回复重定向信息，以便客户端可以和新的master通信。

该命令有两个选项：FORCE和TAKEOVER，下面我们来解释一下这两个选项的作用。

FORCE选项：slave节点不会和master做协商，直接从上述第4步开始进行故障切换

TAKEOVER选项：忽略集群一致验证的人工故障切换。有时会出现集群中master节点不够的情况，此时我们就需要使用TAKEOVER选项将slave批量切换为master节点。

TAKEOVER选项实现了FORCE选项的所有功能，当一个slave节点收到CLUSTER FAILOVER TAKEOVER命令时会有如下操作：

1. 生成一个新的configEpoch，如果本地配置的epoch不是最大的，就需要将其配置为最大。
2. 将原master节点管理的所有slot分配给自己，同时尽快分发最新的配置给所有可达节点。

#### CLUSTER FORGET

最早可用版本：3.0.0

时间复杂度：O(1)

这个命令用于移除指定node-id的node。如果一个node属于某个集群，那么集群中其他节点都会知道它的存在，因此CLUSTER FORGET命令会发送给集群中剩下的所有节点。

然而这个命令并不是简单的把node从node表中删除，它还禁止这个节点再次被添加进集群。

假设我们有4个节点：A、B、C、D，此时我们删除D，如果不把D加入禁用列表，就会发生以下情况：

1. 把D负责的slot重新分配给A、B、C
2. 此时D没有负责的slot了，但仍然在node表中
3. 给A发送命令CLUSTER FORGET D
4. B给A发送一个心跳包，其中包含了D的信息
5. A不知道D的信息，就又开始联系D
6. D又被重新加入到A的node表中

这样我们的删除命令就无效了，除非我们在各个节点没有互相发送心跳包的时候同时给他们发送CLUSTER FORGET命令。但这显然不合理，所以我们实际上应该这样做：

1. 指定的节点从node表中删除
2. 被删除的节点的node-id加入禁用列表一分钟
3. node在处理心跳消息时会忽略禁用列表中的所有node-id的节点

在一些特殊情况下，这个命令会无法执行并返回一个错误。

1. 指定的node-id没有在node表中
2. 收到命令的节点是从节点，而要删除的节点是它的主节点
3. 收到命令的节点和待删除的节点是同一个节点

#### CLUSTER GETKEYSINSLOT

最早可用版本：3.0.0

时间复杂度：O(log(N))，N是请求的key的数量

用法：CLUSTER GETKEYSINSLOT slot count

这个命令返回连接节点指定的slot里key的列表。key的最大数量由count指定。所以这个API可以用作key的批处理。这个命令的主要用途是在做rehash的过程中，把slot从一个节点移动到另外一个节点。

#### CLUSTER INFO

最早可用版本：3.0.0

时间复杂度：O(1)

提供Redis集群的相关信息。

``` bash
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:2
cluster_stats_messages_sent:1483972
cluster_stats_messages_received:1483968
```

- cluster_state：ok状态表示节点可以接收查询请求，fail表示至少有一个slot没有分配或者在error状态。
- cluster_slots_assigned：已经分配到集群节点的slot。16384个slot全部被分配到集群节点是集群节点正常运行的必要条件
- cluster_slots_ok：slot不是FAIL或PFAIL状态的数量
- cluster_slots_pfail：slot状态是PFAIL的数量
- cluster_slots_fail：slot状态是FAIL的数量，如果不是0，那么集群节点将无法提供服务，除非cluster-require-full-coverage被设置为no
- cluster_known_nodes：集群中的节点数量
- cluster_size：至少包含一个slot且能够提供服务的master节点数量
- cluster_current_epoch：集群本地Current Epoch的值
- cluster_my_epoch：当前正在使用节点的Config Epoch值
- cluster_stats_messages_sent：通过点到点总线发送消息的数量
- cluster_stats_messages_received：通过点到点总线接收消息的数量

#### CLUSTER KEYSLOT

最早可用版本：3.0.0

时间复杂度：O(N)，N是key的字节数

返回一个整数，用于标识指定键所散列到的slot。这个命令主要是用于调试和测试。因为它通过一个API来暴露Redis底层hash算法的实现。

#### CLUSTER MEET

最早可用版本：3.0.0

时间复杂度：O(1)

CLUSTER MEET命令用来连接不同Redis节点，以进入工作集群。

有一点基本的共识是节点之间互相都是不信任的，并且被认为是未知节点。以避免因为系统管理错误或者网络地址被修改而导致多个集群的节点混合成一个。

因此，为了使给定的节点能接收另一个节点到Redis Cluster中，有两种方法：

1. 系统管理员发送CLUSTER MEET命令强制一个节点会面另一个节点
2. 一个已知节点发送一个保存在gossip部分的节点列表，包含着未知节点。如果接收的节点已经将发送节点标记为已知节点，那么它会处理gossip中的位置节点信息，并给它发送一个握手消息。

Redis Cluster是一个完整的网络，在创建网络时，并不需要给所有节点发送CLUSTER MEET命令，只要发送了足够的命令，保证每个节点都有已知节点，其他的事情就交给gossip来处理了。

#### CLUSTER NODES

最早可用版本：3.0.0

时间复杂度：O(N)，N是集群中的节点数

该命令提供了当前连接节点所属集群的配置信息。信息格式和Redis集群在磁盘上存储使用的序列化格式完全一样。

通常，如果你想知道hash slot与节点的关联关系，你应该使用CLUSTER SLOTS命令。CLUSTER NODES主要用于管理任务，调试和配置监控。redis-trib也会使用该命令管理集群。

命令的结构如下：

``` bash
<id> <ip:port> <flags> <master> <ping-sent> <pong-recv> <config-epoch> <link-state> <slot> <slot> ... <slot>
```

#### CLUSTER REPLICAS

最早可用版本：5.0.0

时间复杂度：O(1)

该命令会列出主节点的从节点列表。输出格式与CLUSTER NODES格式相同。

若特定节点状态未知，或在接收命令节点不是主节点，则命令失败。

#### CLUSTER REPLICATE

最早可用版本：3.0.0

时间复杂度：O(1)

该命令重新配置一个节点成为指定master的从节点。如果收到命令的节点是empty master，那么该节点的角色将由master转换为slave。

一旦一个节点变成另一个master的slave，不需要将这一变化告知集群内的其他节点，心跳消息会把最新配置同步给其他节点。

一个从节点接收这个命令需要满足以下条件：

1. 指定节点存在它的节点列表中
2. 指定节点对接收命令的节点未知
3. 指定节点是master

如果收到命令的节点不是slave而是master，只有在如下情况下，命令才会执行成功：

1. 该节点不保存任何hash slot
2. 该节点是空的，key空间没有任何key

#### CLUSTER RESET

最早可用版本：3.0.0

时间复杂度：O(N)，N是已知节点的数量

根据reset类型（hard或soft）重置一个集群的节点。当主节点hold住一个或多个key时，这个命令无法执行，必须先使用FLUSHALL命令删除所有的key。该命令的影响是：

1. 集群中的节点都被忽略
2. 所有已分配的slot会被reset，slots-to-nodes关系被完全清除
3. 如果节点是slave，它会被切换成空master。
4. Hard模式：生成新的节点ID
5. Hard模式：currentEpoch和configEpoch被置为0
6. 新配置被持久化到节点磁盘上的集群配置信息文件中

#### CLUSTER SAVECONFIG

最早可用版本：3.0.0

时间复杂度：O(1)

强制保存配置nodes.conf到磁盘。该命令主要用于nodes.conf文件丢失或删除时重新生成文件。

#### CLUSTER SET-CONFIG-EPOCH

最早可用版本：3.0.0

时间复杂度：O(1)

该命令为一个全新的节点设置config epoch，只在以下情况有效：

1. 节点的节点信息表是空的
2. 节点的config epoch是0

人工修改一个节点的config epoch是不安全的，但是当epoch产生冲突时，自动解决又非常慢，这时可以使用这个命令进行人工干预。

#### CLUSTER SETSLOT

最早可用版本：3.0.0

时间复杂度：O(1)

用法：CLUSTER SETSLOTslot IMPORTING|MIGRATING|STABLE|NODE [node-id]

CLUSTER SETSLOT根据子命令修改节点的hash slot状态：

1. MIGRATING：将一个hash slot设置为migrating状态
2. IMPORTING：将一个hash slot设置为importing状态
3. STABLE：清除migrating或importing状态
4. NODE：把hash slot绑定到其他节点

该命令通常在rehash时使用，将源节点的hash slot置为migrating状态，目标节点的hash slot置为importing状态。

- CLUSTER SETSLOT <slot> MIGRATING <destination-node-id>

该命令将slot设置为migrating状态，接下来要处理的key如果存在，命令正常执行。如果不存在，则节点发出ASK重定向，让客户端去请求destination-node节点。对于批量key处理，如果只有部分节点存在，则返回TRYAGAIN错误。

- CLUSTER SETSLOT <slot> IMPORTING <source-node-id>

这是MIGRATING的反操作，接下来涉及该slot的命令都被拒绝，并产生一个MOVED重定向，除非命令跟着一个ASK重定向。

#### CLUSTER SLAVES

最早可用版本：3.0.0

时间复杂度：O(1)

该命令会列出指定master节点的所有slave节点，格式和CLUSTER NODES相同。当指定节点未知或不是master时，命令返回一个错误。

#### CLUSTER SLOTS

最早可用版本：3.0.0

时间复杂度：O(N)，N是slot的总数

该命令返回slot和Redis实例的映射关系。这个命令对客户端很有用，在执行命令时，客户端会根据这个命令返回的信息去连接正确的节点执行命令。

每个节点的信息结构如下：

- 起始slot编号
- 结束slot编号
- slot对应的master节点，用IP/Port表示
- master节点的第一个副本
- 第二个副本

#### READONLY

最早可用版本：3.0.0

时间复杂度：O(1)

开启与Redis Cluster从节点连接的读请求

通常从节点将重定向客户端到认证过的主节点，以获取在指定命令中所涉及的slot，然而客户端可以通过READONLY命令将从节点设置为只读模式。

READONLY告诉Redis Cluster从节点愿意读取可能过时的数据。

当连接处于只读模式，只有操作涉及到该从节点的主节点不服务的键时，集群将会发送一个重定向给客户端，这可能是因为：

1. 客户端发送一个有关这个从节点的主节点不服务hash slot的命令
2. 集群被重新配置并且从节点不在服务给定hash slot的命令

#### READWRITE

最早可用版本：3.0.0

时间复杂度：O(1)

禁止与Redis Cluster从节点连接的读请求。

默认情况下禁止Redis Cluster从节点的读请求，但是可以使用READONLY去在每一个连接的基础上改变这个行为，READWRITE命令将连接的只读模式重置为读写模式。