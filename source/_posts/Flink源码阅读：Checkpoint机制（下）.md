---
title: Flink源码阅读：Checkpoint机制（下）
date: 2025-12-12 23:15:16
tags: Flink
---

书接上回，前文我们梳理的 Checkpoint 机制的源码，但是对于如何写入状态数据并没有深入了解。今天就一起来梳理一下这部分代码。<!-- more -->

### 写在前面

前面我们了解到在 `StreamOperatorStateHandler.snapshotState` 方法中会创建四个 Future，用来支持不同类型的状态写入。

```java
snapshotInProgress.setKeyedStateRawFuture(snapshotContext.getKeyedStateStreamFuture());
snapshotInProgress.setOperatorStateRawFuture(
        snapshotContext.getOperatorStateStreamFuture());

if (null != operatorStateBackend) {
    snapshotInProgress.setOperatorStateManagedFuture(
            operatorStateBackend.snapshot(
                    checkpointId, timestamp, factory, checkpointOptions));
}

if (useAsyncState && null != asyncKeyedStateBackend) {
    if (isCanonicalSavepoint(checkpointOptions.getCheckpointType())) {
        throw new UnsupportedOperationException("Not supported yet.");
    } else {
        snapshotInProgress.setKeyedStateManagedFuture(
                asyncKeyedStateBackend.snapshot(
                        checkpointId, timestamp, factory, checkpointOptions));
    }
}
```

我们主要关心 ManagedState，ManagedState 都是调用 `Snapshotable.snapshot` 方法来写入数据的，下面具体看 KeyedState 和 OperatorState 的具体实现。

### KeyedState

KeyedState 我们以 HeapKeyedStateBackend 为例，
