---
title: Flink源码阅读：状态管理
date: 2025-12-03 11:11:28
tags: Flink
---

前面我们介绍了 Flink 状态的分类和应用。今天从源码层面再看一下 Flink 是如何管理状态的。<!-- more -->

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

这里的创建过程也比较简单，先是获取 KeyGroupRange，它表示的是当前 Operator 上处理的 key 的范围。然后就是创建 StateBackend 实例，这里通过 BackendRestorerProcedure 封装统一的恢复、异常处理和资源清理逻辑。operatorStateBackend 方法的逻辑相比较来说，只是少了 KeyGroupRange 的处理，直接创建 StateBackend 实例。

### 创建和使用 State

#### 创建 KeyedState

KeyedState 是通过调用 StreamingRuntimeContext.getState 方法获取的。我们先来看完整的调用流程。

![getState](https://res.cloudinary.com/dxydgihag/image/upload/v1765164156/Blog/flink/12/getstate.png)

在调用 getState 这些方法时，都会先调用 keyedStateStore 提供的方法，它是 Flink 提供的一个封装 keyedStateBackend 的接口。调用流程的最后，是调用 keyedStateBackend 中的 createOrUpdateInternalState 方法（这里我们以 HeapStateBackend 为例）。

```java
public <N, SV, SEV, S extends State, IS extends S> IS createOrUpdateInternalState(
        @Nonnull TypeSerializer<N> namespaceSerializer,
        @Nonnull StateDescriptor<S, SV> stateDesc,
        @Nonnull StateSnapshotTransformFactory<SEV> snapshotTransformFactory,
        boolean allowFutureMetadataUpdates)
        throws Exception {
    StateTable<K, N, SV> stateTable =
            tryRegisterStateTable(
                    namespaceSerializer,
                    stateDesc,
                    getStateSnapshotTransformFactory(stateDesc, snapshotTransformFactory),
                    allowFutureMetadataUpdates);

    @SuppressWarnings("unchecked")
    IS createdState = (IS) createdKVStates.get(stateDesc.getName());
    if (createdState == null) {
        StateCreateFactory stateCreateFactory = STATE_CREATE_FACTORIES.get(stateDesc.getType());
        if (stateCreateFactory == null) {
            throw new FlinkRuntimeException(stateNotSupportedMessage(stateDesc));
        }
        createdState =
                stateCreateFactory.createState(stateDesc, stateTable, getKeySerializer());
    } else {
        StateUpdateFactory stateUpdateFactory = STATE_UPDATE_FACTORIES.get(stateDesc.getType());
        if (stateUpdateFactory == null) {
            throw new FlinkRuntimeException(stateNotSupportedMessage(stateDesc));
        }
        createdState = stateUpdateFactory.updateState(stateDesc, stateTable, createdState);
    }

    createdKVStates.put(stateDesc.getName(), createdState);
    return createdState;
}


private static final Map<StateDescriptor.Type, StateCreateFactory> STATE_CREATE_FACTORIES =
        Stream.of(
                        Tuple2.of(
                                StateDescriptor.Type.VALUE,
                                (StateCreateFactory) HeapValueState::create),
                        Tuple2.of(
                                StateDescriptor.Type.LIST,
                                (StateCreateFactory) HeapListState::create),
                        Tuple2.of(
                                StateDescriptor.Type.MAP,
                                (StateCreateFactory) HeapMapState::create),
                        Tuple2.of(
                                StateDescriptor.Type.AGGREGATING,
                                (StateCreateFactory) HeapAggregatingState::create),
                        Tuple2.of(
                                StateDescriptor.Type.REDUCING,
                                (StateCreateFactory) HeapReducingState::create))
                .collect(Collectors.toMap(t -> t.f0, t -> t.f1));
```

这里首先是注册了一个 StateTable，这个是 State 中一个非常重要的成员变量，它内部是一个类似 Map 的结构，用来保存 key 和 key 的状态。

STATE_CREATE_FACTORIES 这个变量保存了不同类型的 State 和它对应的创建方法，同理 STATE_UPDATE_FACTORIES 保存的是不同 State 对应的 更新方法。

#### 创建 OperatorState

看完了 KeyedState 的创建过程后，我们再来看下 OperatorState 的创建过程。

OperatorState 的创建方法是通过 FunctionInitializationContext 先获取到 OperatorStateStore，它与 KeyedStateStore 类似，都是对 StateBackend 的方法进行了封装。

```java
@Override
public void initializeState(FunctionInitializationContext context) throws Exception {
    ListStateDescriptor<Tuple2<String, Integer>> descriptor =
            new ListStateDescriptor<>(
                    "buffered-elements",
                    TypeInformation.of(new TypeHint<Tuple2<String, Integer>>() {}));

    checkpointedState = context.getOperatorStateStore().getListState(descriptor);

    if (context.isRestored()) {
        for (Tuple2<String, Integer> element : checkpointedState.get()) {
            bufferedElements.add(element);
        }
    }
}
```

OperatorStateStore 的 getListState 方法中，直接创建出了 PartitionableListState，同时也做了一些缓存操作。

```java
private <S> ListState<S> getListState(
        ListStateDescriptor<S> stateDescriptor, OperatorStateHandle.Mode mode)
        throws StateMigrationException {

    ......
    PartitionableListState<S> partitionableListState =
            (PartitionableListState<S>) registeredOperatorStates.get(name);

    if (null == partitionableListState) {
        // no restored state for the state name; simply create new state holder

        partitionableListState =
                new PartitionableListState<>(
                        new RegisteredOperatorStateBackendMetaInfo<>(
                                name, partitionStateSerializer, mode));

        registeredOperatorStates.put(name, partitionableListState);
    } else {
        ......
    }

    accessedStatesByName.put(name, partitionableListState);
    return partitionableListState;
}
```

PartitionableListState 内部有一个 ArrayList 用于保存数据。

#### 使用 KeyedState

了解完 State 的创建之后，接下来就是 State 的使用了。我们以 HeapValueState 为例来看如何获取 State。

```java
// HeapValueState 类
public V value() {
    final V result = stateTable.get(currentNamespace);

    if (result == null) {
        return getDefaultValue();
    }

    return result;
}
```

在 HeapValueState 类的 value 方法中，直接调用 StateTable 的 get 方法，最终调用的是 CopyOnWriteStateMap 的 get 方法，这个方法与 HashMap 的 get 方法比较类似。

```java
public S get(K key, N namespace) {

    final int hash = computeHashForOperationAndDoIncrementalRehash(key, namespace);
    final int requiredVersion = highestRequiredSnapshotVersion;
    final StateMapEntry<K, N, S>[] tab = selectActiveTable(hash);
    int index = hash & (tab.length - 1);

    for (StateMapEntry<K, N, S> e = tab[index]; e != null; e = e.next) {
        final K eKey = e.key;
        final N eNamespace = e.namespace;
        if ((e.hash == hash && key.equals(eKey) && namespace.equals(eNamespace))) {

            // copy-on-write check for state
            if (e.stateVersion < requiredVersion) {
                // copy-on-write check for entry
                if (e.entryVersion < requiredVersion) {
                    e = handleChainedEntryCopyOnWrite(tab, hash & (tab.length - 1), e);
                }
                e.stateVersion = stateMapVersion;
                e.state = getStateSerializer().copy(e.state);
            }

            return e.state;
        }
    }

    return null;
}
```

#### 使用 OperatorState

OperatorState 底层使用的是 PartitionableListState，前面也提到了，它的内部用了一个 ArrayList 来保存数据，对于 OperatorState 的各种操作也都是来操作这个 ArrayList。

```java
@Override
public void clear() {
    internalList.clear();
}

@Override
public Iterable<S> get() {
    return internalList;
}

@Override
public void add(S value) {
    Preconditions.checkNotNull(value, "You cannot add null to a ListState.");
    internalList.add(value);
}

@Override
public void update(List<S> values) {
    internalList.clear();

    addAll(values);
}

@Override
public void addAll(List<S> values) {
    Preconditions.checkNotNull(values, "List of values to add cannot be null.");
    if (!values.isEmpty()) {
        for (S value : values) {
            checkNotNull(value, "Any value to add to a list cannot be null.");
            add(value);
        }
    }
}
```

### 总结

本文对 State 的相关代码进行了梳理。包括 StateBackend 的创建，KeyedState 和 OperatorState 的创建和使用。State 和 Checkpoint 两者需要结合使用，因此后面我们会再梳理 Checkpoint 的相关代码。
