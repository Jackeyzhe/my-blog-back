---
title: Flink源码阅读：Netty通信
date: 2026-01-05 10:23:02
tags: Flink
---

前文中我们了解了 Flink 的数据交互过程，上游的 Task 将数据写入到 ResultSubpartition 的 buffers 队列中。下游的 Task 通过 LocalInputChannel 和 RemoteInputChannel 消费上游的数据。<!-- more -->

LocalInputChannel 是上下游的 Task 部署在同一个 TaskManager 时使用的，在本地即可完成数据交换，无需网络通信。当上下游的 Task 部署在不同的 TaskManager 时，就需要用到 RemoteInputChannel，Flink 利用 Netty 来进行数据交互。本文我们来一起梳理一下 Netty 相关的源码。

### 初始化

我们先来看 NettyServer 和 NettyClient 的初始化过程。

![NettyInit](https://res.cloudinary.com/dxydgihag/image/upload/v1767601135/Blog/flink/19/NettyInit.png)

Netty 的初始化阶段是在 TaskManager 启动的过程中执行的。在 `TaskManagerServices.fromConfiguration` 方法中，会创建并启动 ShuffleEnvironment。

```java
public static TaskManagerServices fromConfiguration(
        TaskManagerServicesConfiguration taskManagerServicesConfiguration,
        PermanentBlobService permanentBlobService,
        MetricGroup taskManagerMetricGroup,
        ExecutorService ioExecutor,
        ScheduledExecutor scheduledExecutor,
        FatalErrorHandler fatalErrorHandler,
        WorkingDirectory workingDirectory)
        throws Exception {
    ...

    final ShuffleEnvironment<?, ?> shuffleEnvironment =
            createShuffleEnvironment(
                    taskManagerServicesConfiguration,
                    taskEventDispatcher,
                    taskManagerMetricGroup,
                    ioExecutor,
                    scheduledExecutor);
    final int listeningDataPort = shuffleEnvironment.start();
    ...
}
```

我们顺着调用链路可以一直找到 `NettyShuffleServiceFactory.createNettyShuffleEnvironment` 方法，这个方法中创建了 NettyConnectionManager，在 NettyConnectionManager 中有几个很重要的对象。

```java
public NettyConnectionManager(
        NettyBufferPool bufferPool,
        ResultPartitionProvider partitionProvider,
        TaskEventPublisher taskEventPublisher,
        NettyConfig nettyConfig,
        boolean connectionReuseEnabled) {

    this.server = new NettyServer(nettyConfig);
    this.client = new NettyClient(nettyConfig);
    this.bufferPool = checkNotNull(bufferPool);

    this.partitionRequestClientFactory =
            new PartitionRequestClientFactory(
                    client, nettyConfig.getNetworkRetries(), connectionReuseEnabled);

    this.nettyProtocol =
            new NettyProtocol(
                    checkNotNull(partitionProvider), checkNotNull(taskEventPublisher));
}
```

server 和 client 不需要多介绍，就是 Netty 的服务端和客户端。bufferPool 是缓冲池，用于存储要传输的数据。nettyProtocol 提供了 NettyClient 和 NettyServer 引导启动注册的 Channel Handler。

后面就是创建 NettyShuffleEnvironment 及其需要的对象了。在创建完成后，会调用它的 start 方法启动。这个启动方法就是调用了 `connectionManager.start`，在 NettyConnectionManager 中，就是初始化客户端和服务端。

```java
public int start() throws IOException {
    client.init(nettyProtocol, bufferPool);

    return server.init(nettyProtocol, bufferPool);
}
```

#### client 初始化

client 的初始化过程是先创建并初始化 Bootstrap。

```java
private void initEpollBootstrap() {
    // Add the server port number to the name in order to distinguish
    // multiple clients running on the same host.
    String name =
            NettyConfig.CLIENT_THREAD_GROUP_NAME + " (" + config.getServerPortRange() + ")";

    EpollEventLoopGroup epollGroup =
            new EpollEventLoopGroup(
                    config.getClientNumThreads(), NettyServer.getNamedThreadFactory(name));
    bootstrap.group(epollGroup).channel(EpollSocketChannel.class);

    config.getTcpKeepIdleInSeconds()
            .ifPresent(idle -> bootstrap.option(EpollChannelOption.TCP_KEEPIDLE, idle));
    config.getTcpKeepInternalInSeconds()
            .ifPresent(
                    interval -> bootstrap.option(EpollChannelOption.TCP_KEEPINTVL, interval));
    config.getTcpKeepCount()
            .ifPresent(count -> bootstrap.option(EpollChannelOption.TCP_KEEPCNT, count));
}
```

初始化过程重要设置 EventLoopGroup 和 channel，可以用 epoll 的话就用 epoll，否则就用 nio。设置好这些后就是设置了一些通道参数（连接超时时间、Bufffer 池等）。

到这里 client 的初始化其实并没有结束，还需要设置 Handler 流水线，这些工作是在 Task 启动时执行了。

#### server 初始化

server 的初始化过程是先创建并初始化了 ServerBootstrap。之后同样也是设置 EventLoopGroup 和 channel，以及通道相关的各种参数。

设置好之后，会添加 ChannelHandler 流水线，这里的 ChannelHandler 流水线就是我们前面创建的 NettyProtocol 提供的。

```java
public ChannelHandler[] getServerChannelHandlers() {
    PartitionRequestQueue queueOfPartitionQueues = new PartitionRequestQueue();
    PartitionRequestServerHandler serverHandler =
            new PartitionRequestServerHandler(
                    partitionProvider, taskEventPublisher, queueOfPartitionQueues);

    return new ChannelHandler[] {
        messageEncoder,
        new NettyMessage.NettyMessageDecoder(),
        serverHandler,
        queueOfPartitionQueues
    };
}
```

流水线上包含了消息编码器、解码器、PartitionRequestServerHandler 请求服务端处理器和 PartitionRequestQueue 分区请求队列。

这些都设置好之后，就开始启动 NettyServer 服务了。
