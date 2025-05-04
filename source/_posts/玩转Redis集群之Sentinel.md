---
title: 玩转Redis集群之Sentinel
date: 2018-11-04 17:08:45
tags: Redis
---

Redis作为内存数据库，需要具备高可用的特点，不然如果服务器宕机，还在内存里的数据就会丢失。我们最常用的高可用方法就是搭建集群，master机器挂了，可以让slave机器顶上，继续提供服务。但是Redis集群是不会自动进行主从切换的，也就是说，如果主节点非常不争气的在凌晨3点挂了，那么运维同学就要马上起床，把从节点改成主节点，这样的操作是非常繁琐低效的。为此，Redis官方提供了一种解决方案：Redis Sentinel<!-- more -->

#### 简介

Redis Sentinel集群通常由3到5个节点组成，如果个别节点挂了，集群还可以正常运作。它负责监控Redis集群的健康情况。如果主节点挂掉，Sentinel集群会通过投票选择一个新的主节点。当原来的主节点恢复时，它会被当做新的主节点的从节点重新加入Redis集群。

#### 基本原理

Sentinel集群通过指定的配置文件发现master，对其进行监控，并且会发送info指令获取master的从节点信息。Sentinel集群中的节点通过向其监控的主从节点发送hello信息（包含Sentinel本身的ip、端口和id等内容）来向其他Sentinel宣告自己的存在。

Sentinel集群通过订阅连接来接收其他Sentinel的hello信息。

Sentinel集群通过ping命令来检查监控的实例状态，如果在指定时间内没有返回，则认为该实例下线。

Sentinel触发failover主从切换后，并不会马上进行，只有指定(quorum)Sentinel授权后，master节点被标记为ODOWN状态。这时才真正开始投票选择新的master。

Sentinel选择新的master的原则是：首先判断优先级，选择优先级较小的；如果优先级相同，查看复制下标，选择复制数据较多的；如果复制下标也相同，就选择进程ID较小的。

Sentinel被授权后，它将会获得宕掉的master的一份最新配置版本号(config-epoch)，当failover执行结束以后，这个版本号将会被用于最新的配置，通过广播形式通知其它Sentinel，其它的Sentinel则更新对应master的配置。

#### 基本使用

我们以Python为例，简单说明一下在客户端如何使用Sentinel

``` python
from redis.sentinel import Sentinel

if __name__ == '__main__':
    sentinel = Sentinel(['localhost', 26379], socket_timeout=0.1)
    print(sentinel.discover_master('mymaster'))
    print(sentinel.discover_slaves('mymaster'))
    master = sentinel.master_for('mymaster', socket_timeout=0.1)
    slave = sentinel.slave_for('mymaster', socket_timeout=0.1)
    master.set('follow', 'Jackeyzhe2018')
    follow = slave.get('follow')
    print(follow)
```

master_for和slave_for方法会从连接池中拿出一个连接来使用，如果从地址有多个，则会采用轮询的方法。

当redis发生了主从切换时，客户端如何知道地址已经变更了呢？我们从redis-py的源码里找一找答案。

![redis-py](https://res.cloudinary.com/dxydgihag/image/upload/v1541434700/Blog/Redis/redis-py.png)

![get_master_address](https://res.cloudinary.com/dxydgihag/image/upload/v1541434964/Blog/Redis/redis-py2.png)

可以看到，redis在创建一个新的连接时，会调用get_master_address方法来获取主节点地址。get_master_address方法中，客户端先查询主节点地址，然后与内存中的地址进行比较。如果不一致，则会断开连接，然后使用新的地址重新进行连接。

如果主节点没有挂，而Sentinel主动进行了主从切换，对于这种情况redis-py也做了处理。就是捕获一个ReadOnlyError的异常，然后断开连接，后续指令都需要重新进行连接了。当然，如果没有修改性指令，那么连接就不会切换，不过数据也不会被破坏，所以影响不大。

#### 动手搭建

关于Sentinel的工作原理和使用方法我们已经有了大概的认识，为了加深理解，我们来自己动手搭建一套Sentinel集群。

首先搭建我们我需要的redis集群环境

安装好redis后，将redis目录下的配置文件redis.conf复制3份。分别命名为redis6379.conf，redis6380.conf，redis6381.conf。

在redis6381.conf文件中修改以下几项

``` bash
bind 127.0.0.1
port 6381
logfile "6381.log"
dbfilename "dump-6381.rdb"
```

在redis6379.conf中修改

``` bash
bind 127.0.0.1
port 6379
logfile "6379.log"
dbfilename "dump-6379.rdb"
slaveof 127.0.0.1 6381
```

redis6380.conf的修改参照redis6379.conf。修改完成后，分别启动三个实例。就搭建好了我们想要的redis主从环境了。

![master-slave](https://res.cloudinary.com/dxydgihag/image/upload/v1541481896/Blog/Redis/redis-group.png)

我们连接上master节点，可以看到它的主从配置信息

![master](https://res.cloudinary.com/dxydgihag/image/upload/v1541481909/Blog/Redis/redis-master.png)

接着，我们来配置Sentinel集群。这里我们同样配置三个实例。复制3份sentinel.conf文件，分别命名为sentinel-26379.conf，sentinel-26380.conf和sentinel-26381.conf。

sentinel-26379.conf文件中编辑以下内容

``` bash
port 26379  
daemonize yes  
logfile "26379.log"  
dir /home/xxx/redis/data  
sentinel monitor mymaster 127.0.0.1 6381 2
sentinel down-after-milliseconds mymaster 30000  
sentinel parallel-syncs mymaster 1  
sentinel failover-timeout mymaster 180000
```

sentinel-26380.conf和sentinel-26381.conf的内容与上述类似。配置好后，我们使用命令redis-sentinel来启动3个sentinel实例。

![redis-sentinel](https://res.cloudinary.com/dxydgihag/image/upload/v1541481923/Blog/Redis/redis-sentinel.png)

此时，我们用redis-cli命令连接26379的实例，查看sentinel的信息。

![sentinel-info](https://res.cloudinary.com/dxydgihag/image/upload/v1541481877/Blog/Redis/info-sentinel.png)

发现它已经开始监控我们的3个redis节点了。这时我们的整个集群就部署好了，接下来测试一下。

kill掉master节点，查看sentinel的日志，会发现sentinel已经按照我们前面说的步骤选择了新的master。

![sentinel-log](https://res.cloudinary.com/dxydgihag/image/upload/v1541481931/Blog/Redis/sentinel-log.png)

此时再来看sentinel信息。

![new-master](https://res.cloudinary.com/dxydgihag/image/upload/v1541481866/Blog/Redis/after-vote.png)

此时，6380已经成了新的master。

恭喜你，以后都不需要在凌晨起床切换Redis主从实例了。