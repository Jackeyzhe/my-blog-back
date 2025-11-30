---
title: Flink源码阅读：集群启动
date: 2025-11-27 16:35:40
tags: Flink
---

前文中，我们已经了解了 Flink 的三种执行图是怎么生成的。今天继续看一下 Flink 集群是如何启动的。<!-- more -->

### 启动脚本

集群启动脚本的位置在：

```bash
flink-dist/src/main/flink-bin/bin/start-cluster.sh
```

脚本会负责启动 JobManager 和 TaskManager，我们主要关注 standalone 启动模式，具体的流程见下图。

![start-cluster](https://res.cloudinary.com/dxydgihag/image/upload/v1764258366/Blog/flink/11/start-cluster.png)

从图中可以看出 JobManager 是通过 jobmanager.sh 文件启动的，TaskManager 是通过taskmanager.sh 启动的，两者都调用了 flink-daemon.sh，通过传递不同的参数，最终运行不同的 Java 类。

```bash
case $DAEMON in
    (taskexecutor)
        CLASS_TO_RUN=org.apache.flink.runtime.taskexecutor.TaskManagerRunner
    ;;

    (zookeeper)
        CLASS_TO_RUN=org.apache.flink.runtime.zookeeper.FlinkZooKeeperQuorumPeer
    ;;

    (historyserver)
        CLASS_TO_RUN=org.apache.flink.runtime.webmonitor.history.HistoryServer
    ;;

    (standalonesession)
        CLASS_TO_RUN=org.apache.flink.runtime.entrypoint.StandaloneSessionClusterEntrypoint
    ;;

    (standalonejob)
        CLASS_TO_RUN=org.apache.flink.container.entrypoint.StandaloneApplicationClusterEntryPoint
    ;;

    (sql-gateway)
        CLASS_TO_RUN=org.apache.flink.table.gateway.SqlGateway
        SQL_GATEWAY_CLASSPATH="`findSqlGatewayJar`":"`findFlinkPythonJar`"
    ;;

    (*)
        echo "Unknown daemon '${DAEMON}'. $USAGE."
        exit 1
    ;;
esac
```

### JobManager 启动流程

在 StandaloneSessionClusterEntrypoint 的 main 方法中，主要就是加载各种配置和环境变量，然后调用 ClusterEntrypoint.runClusterEntrypoint 来启动集群。跟着调用链一直找到 ClusterEntrypoint.runCluster 方法，这里会启动 ResourceManager、DispatcherRunner 等组件。

```java
private void runCluster(Configuration configuration, PluginManager pluginManager)
        throws Exception {
    synchronized (lock) {
        // 初始化各种服务
        initializeServices(configuration, pluginManager);

        // 创建 DispatcherResourceManagerComponentFactory，
        // 包含了三个核心组件的 Factory
        // DispatcherRunnerFactory、ResourceManagerFactory、RestEndpointFactory
        final DispatcherResourceManagerComponentFactory
                dispatcherResourceManagerComponentFactory =
                        createDispatcherResourceManagerComponentFactory(configuration);

        // 启动 ResourceManager、DispatcherRunner、WebMonitorEndpoint
        clusterComponent =
                dispatcherResourceManagerComponentFactory.create(
                        configuration,
                        resourceId.unwrap(),
                        ioExecutor,
                        commonRpcService,
                        haServices,
                        blobServer,
                        heartbeatServices,
                        delegationTokenManager,
                        metricRegistry,
                        executionGraphInfoStore,
                        new RpcMetricQueryServiceRetriever(
                                metricRegistry.getMetricQueryServiceRpcService()),
                        failureEnrichers,
                        this);

        // 关闭服务
        clusterComponent
                .getShutDownFuture()
                .whenComplete(
                        (ApplicationStatus applicationStatus, Throwable throwable) -> {
                            if (throwable != null) {
                                shutDownAsync(
                                        ApplicationStatus.UNKNOWN,
                                        ShutdownBehaviour.GRACEFUL_SHUTDOWN,
                                        ExceptionUtils.stringifyException(throwable),
                                        false);
                            } else {
                                // This is the general shutdown path. If a separate more
                                // specific shutdown was
                                // already triggered, this will do nothing
                                shutDownAsync(
                                        applicationStatus,
                                        ShutdownBehaviour.GRACEFUL_SHUTDOWN,
                                        null,
                                        true);
                            }
                        });
    }
}
```

下面来详细看一下这几个方法， initializeServices 就是负责初始化各种服务，有几个比较重要的可以着重关注下：

```java
// 初始化并启动一个通用的 RPC Service
commonRpcService = RpcUtils.createRemoteRpcService(...);

// 创建一个 IO 线程池，线程数量位 CPU 核数 * 4
ioExecutor = Executors.newFixedThreadPool(...);

// 创建 HA 服务组件，根据配置初始化 Standalone、ZK、K8S 三种
haServices = createHaServices(configuration, ioExecutor, rpcSystem);

// 创建并启动 blobServer,blobServer 可以理解为是 Flink 内部的
blobServer = BlobUtils.createBlobServer(...);
blobServer.start();

// 创建心跳服务
heartbeatServices = createHeartbeatServices(configuration);

// 创建一个监控服务
processMetricGroup = MetricUtils.instantiateProcessMetricGroup(...);
```

createDispatcherResourceManagerComponentFactory 这个方法就是创建了三个工厂类，不需要过多介绍。我们重点关注 dispatcherResourceManagerComponentFactory.create 方法，即 ResourceManager、DispatcherRunner、WebMonitorEndpoint 是如何启动的。

#### WebMonitorEndpoint

WebMonitorEndpoint 的启动流程图如下，图中细箭头代表同一个方法中顺序调用，粗箭头代表进入上一个方法内部的调用。

![webMonitorEndpoint](https://res.cloudinary.com/dxydgihag/image/upload/v1764428386/Blog/flink/11/webmonitor.png)

第一步是通过工厂创建出了 WebMonitorEndpoint，这里就是比较常规的初始化操作。

第二步是调用 WebMonitorEndpoint 的 start 方法开始启动，start 方法内部先是创建了一个 Router 并调用 initializeHandlers 创建了一大堆 handler（是真的一大堆，这个方法有接近一千行，都是在创建 handler），创建完成之后，对 handler 进行排序和去重，再把它们都注册到 Router 中。这里排序是为了确保路由匹配的正确性，排序规则是先静态路径（/jobs/overview），后动态路径（/jobs/:jobid），假如我们没有排序，先注册了 /jobs/:jobid ，后注册 /jobs/overview ，这时当我们请求 /jobs/overview 时，就会被错误的路由到 /jobs/:jobid 上去。

第三步是调用 startInternal 方法，在 startInternal 方法内部只有 leader 选举和启动缓存清理任务两个步骤。

#### ResourceManager

![ResourceManager](https://res.cloudinary.com/dxydgihag/image/upload/v1764517440/Blog/flink/11/resourceManager.png)
