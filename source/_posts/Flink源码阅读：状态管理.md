---
title: Flink源码阅读：状态管理
date: 2025-12-03 11:11:28
tags: Flink
---

前面我们介绍了 Flink 状态的分类和应用。今天从源码层面再看一下 Flink 是如何管理状态的。

### State 概述

关于 State 的详细介绍可以参考 [Flink学习笔记：状态类型和应用](https://jackeyzhe.github.io/2025/08/04/Flink%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9A%E7%8A%B6%E6%80%81%E7%B1%BB%E5%9E%8B%E5%92%8C%E5%BA%94%E7%94%A8/) 和 [Flink学习笔记：状态后端](https://jackeyzhe.github.io/2025/08/24/Flink%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9A%E7%8A%B6%E6%80%81%E5%90%8E%E7%AB%AF/)这两篇文章，为了方面阅读，这里我们再简单介绍一下。

#### State 使用

State 是 Flink 做复杂逻辑所依赖的核心组件。它的分类如下

![State 分类](https://res.cloudinary.com/dxydgihag/image/upload/v1754928263/Blog/flink/4/%E7%8A%B6%E6%80%81%E5%88%86%E7%B1%BB.png)

常见的是 Keyed State 和 Operator State，Keyed State 作用于 KeyedStream 上，Operator State 可以作用于所有的 Operator 上。Keyed State 使用时，需要先创建 StateDescriptor，然后再调用 getState 获取。

```java
ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
        new ValueStateDescriptor<>(
                "average",
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}));
ValueState<Tuple2<Long, Long>> sum = getRuntimeContext().getState(descriptor);
```

Opeartor State 的获取方式与 Keyed State 类似，都需要 StateDescriptor。Operator State 在定义时需要实现 CheckpointedFunction。

#### State 存储

State Backend 用来管理 State 存储，根据存储格式和存储类型的组合，可以分为三类：

1. MemoryStateBackend：HashMapStateBackend 和 JobManagerCheckpointStorage 的组合，即将 State 以 Java 对象的形式存储在 JobManager 内存中。

2. FsStateBackend：HashMapStateBackend 和 FileSystemCheckpointStorage 的组合，将 State 以 Java 对象的形式存储在远端文件系统中。

3. RocksDBStateBackend：EmbeddedRocksDBStateBackend 和 FileSystemCheckpointStorage 的组合，State 序列化后存储在 RocksDB。

### 创建 State Backend

创建 State Backend 的入口在 StreamTask，StreamTask 是 Flink 部署和运行在 TaskManager 的基本单元。

在 StreamTask 的 invoke 方法中，会先调用 restoreStateAndGates 方法去创建 State Backend。完整的调用链路如下图所示。

![stateBackend](https://res.cloudinary.com/dxydgihag/image/upload/v1764852973/Blog/flink/12/statebackend.png)

在 streamOperatorStateContext 方法中，分别调用了 keyedStatedBackend 和 operatorStateBackend 来创建两种 State Backend。

我们先来看 keyedStateBackend 的逻辑。

```java
protected <K, R extends Disposable & Closeable> R keyedStatedBackend(
        TypeSerializer<K> keySerializer,
        String operatorIdentifierText,
        PrioritizedOperatorSubtaskState prioritizedOperatorSubtaskStates,
        CloseableRegistry backendCloseableRegistry,
        MetricGroup metricGroup,
        double managedMemoryFraction,
        StateObject.StateObjectSizeStatsCollector statsCollector,
        KeyedStateBackendCreator<K, R> keyedStateBackendCreator)
        throws Exception {

    if (keySerializer == null) {
        return null;
    }

    ......

    final KeyGroupRange keyGroupRange =
            KeyGroupRangeAssignment.computeKeyGroupRangeForOperatorIndex(
                    taskInfo.getMaxNumberOfParallelSubtasks(),
                    taskInfo.getNumberOfParallelSubtasks(),
                    taskInfo.getIndexOfThisSubtask());

    // Now restore processing is included in backend building/constructing process, so we need
    // to make sure
    // each stream constructed in restore could also be closed in case of task cancel, for
    // example the data
    // input stream opened for serDe during restore.
    CloseableRegistry cancelStreamRegistryForRestore = new CloseableRegistry();
    backendCloseableRegistry.registerCloseable(cancelStreamRegistryForRestore);
    BackendRestorerProcedure<R, KeyedStateHandle> backendRestorer =
            new BackendRestorerProcedure<>(
                    (stateHandles) -> {
                        KeyedStateBackendParametersImpl<K> parameters =
                                new KeyedStateBackendParametersImpl<>(...);
                        return keyedStateBackendCreator.create(...),
                                parameters);
                    },
                    backendCloseableRegistry,
                    logDescription);

    try {
        return backendRestorer.createAndRestore(
                prioritizedOperatorSubtaskStates.getPrioritizedManagedKeyedState(),
                statsCollector);
    } finally {
        if (backendCloseableRegistry.unregisterCloseable(cancelStreamRegistryForRestore)) {
            IOUtils.closeQuietly(cancelStreamRegistryForRestore);
        }
    }
}
```

这里的创建过程也比较简单，先是获取 KeyGroupRange，表示的是
