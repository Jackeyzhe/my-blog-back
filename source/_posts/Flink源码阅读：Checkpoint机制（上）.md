---
title: Flink源码阅读：Checkpoint机制（上）
date: 2025-12-09 15:14:59
tags: Flink
---

前文我们梳理了 Flink 状态管理相关的源码，我们知道，状态是要与 Checkpoint 配合使用的。因此，本文我们就一起来看一下 Checkpoint 相关的源码。<!-- more -->

### 写在前面

在[Flink学习笔记：如何做容错](https://jackeyzhe.github.io/2025/08/17/Flink%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9A%E5%A6%82%E4%BD%95%E5%81%9A%E5%AE%B9%E9%94%99/)一文中，我们介绍了 Flink 的 Checkpoint 机制。Checkpoint 分为 EXACTLY_ONCE 和 AT_LEAST_ONCE 两种模式。

我们一起回顾一下一次完整的 Checkpoint 具体流程：Checkpoint 是由 CheckpointCoordinator 触发，Source 节点收到触发请求后，会将 State 进行持久化，同时向下游发送 Barrier 消息，下游节点收到 Barrier 消息后，也同样对 State 进行持久化和发送 Barrier 消息。当所有节点都完成持久化过程后 CheckpointCoordinator 会将一些元数据进行持久化。

带着这些背景知识，我们再来梳理一下 Checkpoint 相关的代码。

### JobManager 端触发流程

JobManager 在调用 `DefaultExecutionGraphBuilder.buildGraph` 生成 ExecutionGraph 之后，会调用 `executionGraph.enableCheckpointing` 方法来设置 Checkpoint 相关的配置，这个方法中创建了 CheckpointCoordinator 并注册了 CheckpointCoordinatorDeActivator 这个监听，它负责启动和停止 Checkpoint 的调度。

当作业变成 RUNNING 状态时，CheckpointCoordinator 会部署一个定时任务 ScheduledTrigger，这个定时任务就是用来周期性的触发 Checkpoint。

触发 Checkpoint 的核心逻辑在  `CheckpointCoordinator.startTriggeringCheckpoint` 这个方法中。这个方法中使用了多个 CompletableFuture 来完成整个流程的编排。具体流程见下图（图中不同颜色代表着使用不同线程池执行）。

![checkpoint](https://res.cloudinary.com/dxydgihag/image/upload/v1765450882/Blog/flink/13/checkpoint.png)

- checkpointPlanFuture：这是生成 Checkpoint 执行计划的 Future，Checkpoint Plan 中维护了三个关键的集合：tasksToTrigger、tasksToWaitFor 和 tasksToCommitTo。tasksToTrigger 是所有的 Source 节点，表示触发 Checkpoint 的节点，另外两个集合都包含了全部节点，分别表示等待进行 Checkpoint 的节点和等待提交的节点。

- pendingCheckpointCompletableFuture：生成完 Checkpoint Plan 之后，会创建 pendingCheckpointCompletableFuture，这个 Future 中有两个执行任务，分别是生成自增的 CheckpointID 和 创建 PendingCheckpoint。PendingCheckpoint 中维护了等待完成的 task 列表，当所有 task 都确认完成之后，PendingCheckpoint 会变成 CompletedCheckpoint。

- coordinatorCheckpointsComplete：这个 Future 也有两个任务，第一个是初始化存储路径，第二个是触发所有 OperatorCoordinator Checkpoint，并确认它们的状态。

- masterStatesComplete：触发快照所有的 Master Hook，这一步主要是 CheckpointCoordinator 用来收集 JobManager 级别状态。

- masterTriggerCompletionPromise：在 masterStatesComplete 和 coordinatorCheckpointsComplete 都执行完成后，会开始执行 masterTriggerCompletionPromise。masterTriggerCompletionPromise 的任务是调用 triggerCheckpointRequest 来产生 Barrier 消息。具体的触发流程见下图。

![triggerTask](https://res.cloudinary.com/dxydgihag/image/upload/v1765468624/Blog/flink/13/triggerTaskCheckpoint.png)

至此，JobManager 端的触发流程就完成了，接下来就到了 TaskManager 端了。

### TaskManager 端执行流程
