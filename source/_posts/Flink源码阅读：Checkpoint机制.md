---
title: Flink源码阅读：Checkpoint机制
date: 2025-12-09 15:14:59
tags: Flink
---

前文我们梳理了 Flink 状态管理相关的源码，我们知道，状态是要与 Checkpoint 配合使用的。因此，本文我们就一起来看一下 Checkpoint 相关的源码。<!-- more -->

### 写在前面

在[Flink学习笔记：如何做容错](https://jackeyzhe.github.io/2025/08/17/Flink%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9A%E5%A6%82%E4%BD%95%E5%81%9A%E5%AE%B9%E9%94%99/)一文中，我们介绍了 Flink 的 Checkpoint 机制。Checkpoint 分为 EXACTLY_ONCE 和 AT_LEAST_ONCE 两种模式。

我们一起回顾一下一次完整的 Checkpoint 具体流程：Checkpoint 是由 CheckpointCoordinator 触发，Source 节点收到触发请求后，会将 State 进行持久化，同时向下游发送 Barrier 消息，下游节点收到 Barrier 消息后，也同样对 State 进行持久化和发送 Barrier 消息。当所有节点都完成持久化过程后 CheckpointCoordinator 会将一些元数据进行持久化。

带着这些背景知识，我们再来梳理一下 Checkpoint 相关的代码。

### JobManager 端

JobManager 在调用 `DefaultExecutionGraphBuilder.buildGraph` 生成 ExecutionGraph 之后，会调用 `executionGraph.enableCheckpointing` 方法来设置 Checkpoint 相关的配置，这个方法中创建了 CheckpointCoordinator 并注册了 CheckpointCoordinatorDeActivator 这个监听，它负责启动和停止 Checkpoint 的调度。

当作业变成 RUNNING 状态时，CheckpointCoordinator 会部署一个定时任务 ScheduledTrigger，这个定时任务就是用来周期性的触发 Checkpoint。

触发 Checkpoint 的核心逻辑在  `CheckpointCoordinator.startTriggeringCheckpoint` 这个方法中。这个方法中使用了多个 CompletableFuture 来完成整个流程的编排。

![checkpoint](https://res.cloudinary.com/dxydgihag/image/upload/v1765450882/Blog/flink/13/checkpoint.png)
