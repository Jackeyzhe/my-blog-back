---
title: 玩转Redis集群之Codis
date: 2018-11-14 23:14:39
tags: Redis
---

近几年，随着互联网的飞速发展，作为程序员，我们需要处理的数据规模也在不断扩大。如果你使用Redis作为数据库时，面临大数据高并发的场景时，单个Redis实例就会显得力不从心。这时Redis的集群方案应运而生，他将众多Redis实例综合起来，共同应对大数据高并发的场景。<!-- more -->

Codis是Redis集群方案的一种。它是由豌豆荚的中间件团队开发的，所以，它有一套详细的中文版README，方便大家学习。

![Codis架构](https://res.cloudinary.com/dxydgihag/image/upload/v1542815602/Blog/Redis/architecture.png)

它的架构如上图所示，由codis-proxy对外提供Redis的服务。ZooKeeper用来存储数据路由表和codis-proxy节点的元信息。codis-proxy会监听所有的redis集群，当Redis集群处理能力达到上限时，可以动态增加Redis实例来实现扩容的需求。



#### 组件介绍

- Codis Proxy：像刚才所说的，它对外提供Redis服务，除了一些不支持的命令外（[不支持的命令列表](https://github.com/CodisLabs/codis/blob/release3.2/doc/unsupported_cmds.md)），表现的和原生的Redis没有区别。由于它是无状态的，所以我们可以部署多个节点，从而保证了可用性。
- Codis Dashboard：集群管理工具，支持Codis Proxy的添加删除以及数据迁移等操作。对于一个Codis集群，Dashboard最多部署一个
- Codis Admin：集群管理的命令行工具
- Codis FE：集群管理界面，多个Codis集群可以共用一个Codis FE，通过配置文件管理后端的codis-dashboard
- Storage：为集群提供外部存储，目前支持ZooKeeper、Etcd、Fs三种。
- Codis Server：基于3.2.8分支开发，增加额外的数据结构，用来支持slot有关的操作及数据迁移指令。



#### Codis分片原理

现在我们已经知道了Codis会将指定key的Redis命令转发给下层的Redis。那么Codis如何知道某个key在哪个Redis上呢。

Codis采用Pre-sharding的技术来实现数据分片，默认分为1024个slot（0-1023）。Codis在接收到命令时，先对key进行[crc32](https://baike.baidu.com/item/CRC32/7460858?fr=aladdin)运算，然后再对1024取余，得到的结果就是对应的slot。然后就可以将命令转发给slot对应的Redis实例进行处理了。



#### 扩容操作

Codis的动态扩容/缩容能力是它的一大亮点之一。它可以对Redis客户端透明。在扩容时，Codis提供了SLOTSSCAN指令，这个指令可以扫描指定的slot上的所有key，然后对每个key进行迁移。在扩容过程中，如果有新的key需要转发到正在迁移的slot上，那么codis会判断这个key是否需要迁移，如果需要，则对指定的key进行强制迁移，迁移完成后，再将命令转发到新的Redis上。

看了上面的介绍是不是觉得扩容是一件很麻烦的事情，Codis已经为我们考虑到这点了，它提供了自动均衡的功能，只需要在界面上点一下"Auto Rebalance"按钮，就可以自动实现slot迁移（可以说非常贴心了）。缩容也比较简单，只需要将需要下线的实例的slot迁移到其他实例上，然后删除group就可以了。



#### Codis的缺点

当Redis Group的master挂掉时，codis不会自动将某个slave升为master，codis提供了一个叫做codis-ha的工具，这个工具通过dashboard提供RESTful API来实现自动主从切换。但是，当codis将某个slave升为master时，其他的slave并不会改变状态，仍然会从旧的master上同步数据，这就导致了主从数据不一致。因此，当出现主从切换时，需要管理员手动创建新的sync action来完成数据同步。

此外，Codis还面临一个比较尴尬的情况就是，由于它不是Redis“亲生”的，因此，当Redis发布了new feature时，它总会慢一步，因此，它需要在Redis发布new feature后迅速赶上，以保持竞争力。



#### 搭建Codis

1. 安装Go运行环境

Mac用户可以[参考这个](https://blog.helloarron.com/2015/08/29/go/mac-install-go/)，其他系统的用户也可以看这个[教程](http://www.runoob.com/go/go-environment.html)。

安装好以后，验证一下是否安装成功

``` bash
$ go version
go version go1.11.2 darwin/amd64
```

2. 下载Codis源码

需要下载到指定目录：$GOPATH/src/github.com/CodisLabs/codis

``` bash
$ mkdir -p $GOPATH/src/github.com/CodisLabs
$ cd $_ && git clone https://github.com/CodisLabs/codis.git -b release3.2
```

3. 编译源码

进入源码的codis目录，直接执行make命令即可。编译完成后，bin目录下的结构应该是这样的

``` bash
$ ll bin 
total 178584
drwxr-xr-x  8 jackey  staff   256B 11 13 10:57 assets
-rwxr-xr-x  1 jackey  staff    17M 11 13 10:57 codis-admin
-rwxr-xr-x  1 jackey  staff    18M 11 13 10:56 codis-dashboard
-rw-r--r--  1 jackey  staff     5B 11 21 18:06 codis-dashboard.pid
-rwxr-xr-x  1 jackey  staff    16M 11 13 10:57 codis-fe
-rw-r--r--  1 jackey  staff     5B 11 21 18:24 codis-fe.pid
-rwxr-xr-x  1 jackey  staff    15M 11 13 10:57 codis-ha
-rwxr-xr-x  1 jackey  staff    19M 11 13 10:57 codis-proxy
-rw-r--r--  1 jackey  staff     5B 11 21 18:08 codis-proxy.pid
-rwxr-xr-x  1 jackey  staff   1.1M 11 13 10:56 codis-server
-rwxr-xr-x  1 jackey  staff    98K 11 13 10:56 redis-benchmark
-rwxr-xr-x  1 jackey  staff   161K 11 13 10:56 redis-cli
-rwxr-xr-x  1 jackey  staff   1.1M 11 13 10:56 redis-sentinel
-rw-r--r--  1 jackey  staff   170B 11 13 10:56 version
```

到这里为止，我们的准备工作已经完成了。接下来我们来看一下如何在单机环境启动测试集群。

4. 启动codis-dashboard

进入admin目录，执行codis-dashboard-admin.sh脚本

``` bash
$ ./codis-dashboard-admin.sh start
/Users/jackey/Documents/go_workspace/src/github.com/CodisLabs/codis/admin/../config/dashboard.toml
starting codis-dashboard ... 
```

然后查看日志，观察是否启动成功

``` bash
$ tail -100 ../log/codis-dashboard.log.2018-11-21
2018/11/21 18:06:57 main.go:155: [WARN] option --pidfile = /Users/jackey/Documents/go_workspace/src/github.com/CodisLabs/codis/bin/codis-dashboard.pid
2018/11/21 18:06:57 topom.go:429: [WARN] admin start service on [::]:18080
2018/11/21 18:06:57 fsclient.go:195: [INFO] fsclient - create /codis3/codis-demo/topom OK
2018/11/21 18:06:58 topom_sentinel.go:169: [WARN] rewatch sentinels = []
2018/11/21 18:06:58 main.go:179: [WARN] [0xc000374120] dashboard is working ...
```

5. 启动codes-proxy

执行codis-proxy-admin.sh脚本

``` bash
$ ./codis-proxy-admin.sh start                   
/Users/jackey/Documents/go_workspace/src/github.com/CodisLabs/codis/admin/../config/proxy.toml
starting codis-proxy ...
```

查看是否启动成功

``` bash
$ tail -100 ../log/codis-proxy.log.2018-11-21
2018/11/21 18:08:34 proxy_api.go:44: [WARN] [0xc0003262c0] API call /api/proxy/start/212d13827c84455d487036d4bb07ce15 from 10.1.201.43:58800 []
2018/11/21 18:08:34 proxy_api.go:44: [WARN] [0xc0003262c0] API call /api/proxy/sentinels/212d13827c84455d487036d4bb07ce15 from 10.1.201.43:58800 []
2018/11/21 18:08:34 proxy.go:293: [WARN] [0xc0003262c0] set sentinels = []
2018/11/21 18:08:34 main.go:343: [WARN] rpc online proxy seems OK
2018/11/21 18:08:35 main.go:233: [WARN] [0xc0003262c0] proxy is working ...
```

6. 启动codis-server

执行codis-server-admin.sh脚本

``` bash
$ ./codis-server-admin.sh start
/Users/jackey/Documents/go_workspace/src/github.com/CodisLabs/codis/admin/../config/redis.conf
starting codis-server ... 
```

查看是否启动成功

``` bash
$ tail -100 /tmp/redis_6379.log
12854:M 21 Nov 18:09:29.172 * Increased maximum number of open files to 10032 (it was originally set to 256).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 3.2.11 (de1ad026/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 12854
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

12854:M 21 Nov 18:09:29.187 # Server started, Redis version 3.2.11
12854:M 21 Nov 18:09:29.187 * The server is now ready to accept connections on port 6379
```

如果执行报错，请先确认使用的用户是否有/tmp/redis_6379.log文件的读写权限。

这里我为了测试Codis的Auto Rebalance功能，所以启动了两个实例。方法很简单，只需要分别将admin/codis-server-admin.sh和config/redis.conf这两个文件复制一份，修改文件中的端口等信息，然后再以同样的方法执行一下新的脚本。

7. 启动codis-fe

执行codis-fe-admin.sh脚本

``` bash
$ ./codis-fe-admin.sh start

starting codis-fe ... 
```

查看是否执行成功

``` bash
$ tail -100 ../log/codis-fe.log.2018-11-21 
2018/11/21 18:24:33 main.go:101: [WARN] set ncpu = 4
2018/11/21 18:24:33 main.go:104: [WARN] set listen = 0.0.0.0:9090
2018/11/21 18:24:33 main.go:120: [WARN] set assets = /Users/jackey/Documents/go_workspace/src/github.com/CodisLabs/codis/bin/assets
2018/11/21 18:24:33 main.go:162: [WARN] set --filesystem = /tmp/codis
2018/11/21 18:24:33 main.go:216: [WARN] option --pidfile = /Users/jackey/Documents/go_workspace/src/github.com/CodisLabs/codis/bin/codis-fe.pid
```

全部启动成功之后，就可以访问http://127.0.0.1:9090，开始设置集群了。

8. 添加group

刚刚我们启动了两个codis-server，因此，我们可以new两个group，然后分别将codis-server加入到两个group中

![codis-group](https://res.cloudinary.com/dxydgihag/image/upload/v1542888435/Blog/Redis/codis-group.png)

9. 初始化slot

一开始所有的slot都是offline状态。

![codis-slot-init](https://res.cloudinary.com/dxydgihag/image/upload/v1542801256/Blog/Redis/codis-slot-init.png)

点击下方的Rebalance All Slots按钮，codis会自动把1024个slot分配给两个group（每个分512个）。

![codis-slot](https://res.cloudinary.com/dxydgihag/image/upload/v1542801268/Blog/Redis/codis-slot.png)

当然，也可以手动分配slot，比如，我们将group-1的10个slot分配给group-2，只需要点击Migrate Some按钮即可。

![codis-slot-manual](https://res.cloudinary.com/dxydgihag/image/upload/v1542888791/Blog/Redis/codis-slot-manual.png)



#### 小结

Codis的动态扩容能力简直好用到爆 ，不过目前也存在一些问题（前面我们也介绍过了）。所以你的集群是否要使用Codis还需要看具体的需求。最后还是要为Codis的开发团队点赞，另外他们还开发出了一套分布式数据库——[TiDB](https://github.com/pingcap/tidb)。有兴趣的同学可以学习一下。