---
title: Flink源码阅读：JobManager的HA机制
date: 2026-01-06 22:15:45
tags: Flink
---

JobManager 在 Flink 集群中发挥着重要的作用，包括任务调度和资源管理等工作。如果 JobManager 宕机，那么整个集群的任务都将失败。为了解决 JobManager 的单点问题，Flink 也设计了 HA 机制来保障整个集群的稳定性。<!-- more -->

### 基本概念

在 JobManager 启动时，调用 `HighAvailabilityServicesUtils.createHighAvailabilityServices` 来创建 HA 服务，HA 依赖的服务都被封装在 HighAvailabilityServices 中。当前 Flink 内部支持两种高可用模式，分别是 ZooKeeper 和 KUBERNETES。

```java
case ZOOKEEPER:
    return createZooKeeperHaServices(configuration, executor, fatalErrorHandler);
case KUBERNETES:
    return createCustomHAServices(
            "org.apache.flink.kubernetes.highavailability.KubernetesHaServicesFactory",
            configuration,
            executor);
```

HighAvailabilityServices 中提供的关键组件包括：

- LeaderRetrievalService：服务发现，用于获取当前 leader 的地址。目前用到服务发现的组件有 ResourceManager、Dispatcher、JobManager、ClusterRestEndpoint。

- LeaderElection：选举服务，从多个候选者中选出一个作为 leader。用到选举服务的同样是 ResourceManager、Dispatcher、JobManager、ClusterRestEndpoint 这四个。

- CheckpointRecoveryFactory：Checkpoint 恢复组件的工厂类，提供了创建 CompletedCheckpointStore 和 CheckpointIDCounter 的方法。CompletedCheckpointStore 是用于存储已完成的 checkpoint 的元信息，CheckpointIDCounter 是用于生成 checkpoint ID。

- ExecutionPlanStore：用于存储执行计划。

- JobResultStore：用于存储作业结果，这里有两种状态，一种是 dirty，表示作业没有被完全清理，另一种是 clean，表示作业清理工作已经执行完成了。

- BlobStore：存储作业运行期间的一些二进制文件。

#### 选举服务

Flink 的选举是依靠 LeaderElection 和 LeaderContender 配合完成的。LeaderElection 是 LeaderElectionService 的代理接口，提供了注册候选者、确认 leader 和 判断候选者是否是 leader 三个接口。LeaderContender 则是用来表示候选者对象。当一个 LeaderContender 当选 leader 后，LeaderElectionService 会为其生成一个 leaderSessionId，LeaderContender 会调用 confirmLeadershipAsync 发布自己的地址。选举服务的具体实现在 LeaderElectionDriver 接口中。

#### 服务发现

服务发现的作用是获取各组件的 leader 地址。服务发现依赖 LeaderRetrievalService 和 LeaderRetrievalListener。LeaderRetrievalService 可以启动一个监听，当有新的 leader 当选时，会调用 LeaderRetrievalListener 的 notifyLeaderAddress 方法。

#### 信息保存

当 leader 发生切换时，新的 leader 需要获取到旧 leader 存储的信息，这就需要旧 leader 把这些信息存在一个公共的存储上。它可以是 ZooKeeper 或 Kubernetes 的存储，也可以是分布式文件系统的存储。

### 基于 ZooKeeper 的 HA
