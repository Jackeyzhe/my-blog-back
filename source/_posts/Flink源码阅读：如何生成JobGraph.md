---
title: Flink源码阅读：如何生成JobGraph
date: 2025-09-18 22:02:40
tags: Flink
---

前文我们介绍了 Flink 的四种执行图，并且通过源码了解了 Flink 的 StreamGraph 是怎么生成的，本文我们就一起来看下 Flink 的另一种执行图——JobGraph 是如何生成的。<!-- more -->

### StreamGraph 和 JobGraph 的区别

在正式开始之前，我们再来回顾一下 StreamGraph 和 JobGraph 的区别。假设我们的任务是建造一座大楼，StreamGraph 就像是设计蓝图，它描述了每个窗户、每根水管的位置和规格，而 JobGraph 像是给到施工队的施工流程图，它描述了每个任务模块，例如先把地基浇筑好，再铺设管线等。总的来说，JobGraph 更偏向执行层面，它是由 StreamGraph 优化而来。

回到 Flink 本身，我们通过一个表格来了解两个图的区别。

|      | StreamGraph        | JobGraph                  |
| ---- | ------------------ | ------------------------- |
| 生成阶段 | 客户端，执行 execute() 时 | 客户端，提交前由 StreamGraph 转换生成 |
| 抽象层级 | 高层逻辑图，直接对应 API     | 优化后的执行图，为调度做准备            |
| 核心优化 | 无                  | 主要是算子链优化                  |
| 节点   | StreamNode         | JobVertex                 |
| 边    | StreamEdge         | JobEdge                   |
| 提交对象 | 无                  | 提交给 JobManager            |
| 包含资源 | 无                  | 包含执行作业所需的 Jar 包、依赖库和资源文件  |

### JobVertex

JobGraph 中的节点是 JobVertex，在 StreamGraph 转换成 JobGraph 的过程中，会将多个节点串联起来，最终生成 JobVertex。

JobVertex包含以下成员变量：

![JobVertex](https://res.cloudinary.com/dxydgihag/image/upload/v1758469798/Blog/flink/9/JobVertex.png)

我们分别看一下这些成员变量及其作用。

#### 1、标识符相关

```java
// JobVertex的id，在作业执行过程中的唯一标识。监控、调度和故障恢复都会使用
private final JobVertexID id;

// operator id列表，按照深度优先顺序存储。operator 的管理、状态分配都会用到
private final List<OperatorIDPair> operatorIDs;
```

#### 2、输入输出相关

```java
// 定义所有的输入边
private final List<JobEdge> inputs = new ArrayList<>();

// 定义所有的输出数据集
private final Map<IntermediateDataSetID, IntermediateDataSet> results = new LinkedHashMap<>();

// 输入分片源，主要用于批处理作业，定义如何将数据分成多个片
private InputSplitSource<?> inputSplitSource;
```

#### 3、执行配置相关

```java
// 并行度，即运行时拆分子任务数量，默认使用全局配置
private int parallelism = ExecutionConfig.PARALLELISM_DEFAULT;

// 最大并行度
private int maxParallelism = MAX_PARALLELISM_DEFAULT;

// 存储运行时实际执行的类，使 Flink 可以灵活处理不同类型的操作符
// 流任务可以是"org.apache.flink.streaming.runtime.tasks.StreamTask"
// 批任务可以是"org.apache.flink.runtime.operators.BatchTask"
private String invokableClassName;

// 自定义配置
private Configuration configuration;

// 是否是动态设置并发度
private boolean dynamicParallelism = false;

// 是否支持优雅停止
private boolean isStoppable = false;
```

#### 4、资源管理相关

```java
// JobVertex 最小资源需求
private ResourceSpec minResources = ResourceSpec.DEFAULT;

// JobVertex 推荐资源需求
private ResourceSpec preferredResources = ResourceSpec.DEFAULT;

// 用于资源优化，运行不同的 JobVertex 的子任务运行在同一个 slot
@Nullable private SlotSharingGroup slotSharingGroup;

// 需要严格共址的 JobVertex 组，每个 JobVertex 的第 n 个子任务运行在同一个 TaskManager
@Nullable private CoLocationGroupImpl coLocationGroup;
```

#### 5、协调器

```java
// 操作符协调器，用于处理全局协调逻辑
private final List<SerializedValue<OperatorCoordinator.Provider>> operatorCoordinators =
            new ArrayList<>();
```

#### 6、显示和描述信息

```java
// JobVertex 的名称
private String name;

// 操作符名称，比如 'Flat Map' 或 'Join'
private String operatorName;

// 操作符的描述，比如 'Hash Join' 或 'Sorted Group Reduce'
private String operatorDescription;

// 提供比 name 更友好的描述信息
private String operatorPrettyName;
```

#### 7、状态和行为标志

```java
// 是否支持同一个子任务并发多次执行
private boolean supportsConcurrentExecutionAttempts = true;

// 标记并发度是否被显式设置
private boolean parallelismConfigured = false;

// 是否有阻塞型输出
private boolean anyOutputBlocking = false;
```

#### 8、缓存数据集

```java
// 存储该 JobVertex 需要消费的缓存中间数据集的 ID，可提高作业执行效率
private final List<IntermediateDataSetID> intermediateDataSetIdsToConsume = new ArrayList<>();
```

### JobEdge

在 StreamGraph 中，StreamEdge 是连接 StreamNode 的桥梁。在 JobGraph 中，与之对应的是 JobEdge，不同点在于  JobEdge 中保存的是输入节点和输出结果。

#### 1、连接关系成员

```java
// 定义数据流向哪个 JobVertex
private final JobVertex target;

// 定义这条边的源数据
private final IntermediateDataSet source;

// 输入类型的编号
private final int typeNumber;

// 多个输入间的键是否相关，如果为 true，相同键的数据在一个输入被分割时，在其他数据对应的记录也会发送到相同的下游节点
private final boolean interInputsKeysCorrelated;

// 同一输入内相同的键是否必须发送到同一下游任务
private final boolean intraInputKeyCorrelated;
```

#### 2、数据分发模式

```java
// 定义数据在并行任务期间的分发模式
// 可能值：
// ALL_TO_ALL：全连接，每个上游子任务连接所有下游任务
// POINTWISE：点对点连接，一对一或一对多的本地连接
private final DistributionPattern distributionPattern;
```

#### 3、数据传输策略

```java
// 是否为广播连接
private final boolean isBroadcast;

// 是否为 forward 连接，forward 连接最高效，直接转发，无需序列化网络传输
private final boolean isForward;

// 数据传输策略名称，用于显示
private String shipStrategyName;
```

#### 4、状态重分布映射器

```java
// 下游状态重分布映射器，当作业扩容时，决定是否重新分配下游算子的持久化状态
private SubtaskStateMapper downstreamSubtaskStateMapper = SubtaskStateMapper.ROUND_ROBIN;

// 上游状态重分布映射器，当作业扩容时，决定是否重新分配上游算子的持久化状态
private SubtaskStateMapper upstreamSubtaskStateMapper = SubtaskStateMapper.ROUND_ROBIN;
```

#### 5、描述和缓存信息

```java
// 预处理操作的名称
private String preProcessingOperationName;

// 操作符级别缓存的描述
private String operatorLevelCachingDescription;
```

### StreamGraph 转换成 JobGraph

现在我们再来看一下 StreamGraph 是如何转换成 JobGraph 的。转换逻辑的入口是 StreamGraph.getJobGraph 方法。它只是调用了 StreamingJobGraphGenerator.createJobGraph，核心逻辑在 createJobGraph 方法中。

```java
private JobGraph createJobGraph() {
    // 预验证，检查 StreamGraph 配置正确性
    preValidate(streamGraph, userClassloader);

    // 【核心】链化操作符
    setChaining();

    if (jobGraph.isDynamic()) {
        // 支持动态扩缩容场景，为动态图设置并行度
        setVertexParallelismsForDynamicGraphIfNecessary();
    }

    // Note that we set all the non-chainable outputs configuration here because the
    // "setVertexParallelismsForDynamicGraphIfNecessary" may affect the parallelism of job
    // vertices and partition-reuse
    final Map<Integer, Map<StreamEdge, NonChainedOutput>> opIntermediateOutputs =
            new HashMap<>();
    // 设置不能链化的输出边
    setAllOperatorNonChainedOutputsConfigs(opIntermediateOutputs, jobVertexBuildContext);
    setAllVertexNonChainedOutputsConfigs(opIntermediateOutputs);

    // 设置物理边连接
    setPhysicalEdges(jobVertexBuildContext);

    // 设置支持并发执行的 JobVertex
    markSupportingConcurrentExecutionAttempts(jobVertexBuildContext);

    // 验证混合 shuffle 模式只在批处理模式下使用
    validateHybridShuffleExecuteInBatchMode(jobVertexBuildContext);

    // 设置 Slot 共享和协同定位
    setSlotSharingAndCoLocation(jobVertexBuildContext);

    // 设置托管内存比例
    setManagedMemoryFraction(jobVertexBuildContext);

    // 为 JobVertex 名称添加前缀
    addVertexIndexPrefixInVertexName(jobVertexBuildContext, new AtomicInteger(0));

    // 设置操作符描述信息
    setVertexDescription(jobVertexBuildContext);

    // Wait for the serialization of operator coordinators and stream config.
    // 序列化操作符协调器和流配置
    serializeOperatorCoordinatorsAndStreamConfig(serializationExecutor, jobVertexBuildContext);

    return jobGraph;
}
```

可以看到，在 createJobGraph 方法中，调用了 setChaining 方法，即进行链化操作。这也是 JobGraph 最核心的优化之一。下面我们来看一下具体怎么做链化。

```java
private void setChaining() {
    // we separate out the sources that run as inputs to another operator (chained inputs)
    // from the sources that needs to run as the main (head) operator.
    final Map<Integer, OperatorChainInfo> chainEntryPoints =
            buildChainedInputsAndGetHeadInputs();
    final Collection<OperatorChainInfo> initialEntryPoints =
            chainEntryPoints.entrySet().stream()
                    .sorted(Comparator.comparing(Map.Entry::getKey))
                    .map(Map.Entry::getValue)
                    .collect(Collectors.toList());

    // iterate over a copy of the values, because this map gets concurrently modified
    for (OperatorChainInfo info : initialEntryPoints) {
        createChain(
                info.getStartNodeId(),
                1, // operators start at position 1 because 0 is for chained source inputs
                info,
                chainEntryPoints,
                true,
                serializationExecutor,
                jobVertexBuildContext,
                null);
    }
}
```

setChaining 方法中主要分为两步，第一步是处理 Source 节点，将可以链化的 Source 和不能链化的 Source 节点分开。先来看如何判断一个 Source 是否可被链化。

```java
public static boolean isChainableSource(StreamNode streamNode, StreamGraph streamGraph) {
    // 最基本的一些判空，输出边数量为1
    if (streamNode.getOperatorFactory() == null
            || !(streamNode.getOperatorFactory() instanceof SourceOperatorFactory)
            || streamNode.getOutEdges().size() != 1) {
        return false;
    }
    final StreamEdge sourceOutEdge = streamNode.getOutEdges().get(0);
    final StreamNode target = streamGraph.getStreamNode(sourceOutEdge.getTargetId());
    final ChainingStrategy targetChainingStrategy =
            Preconditions.checkNotNull(target.getOperatorFactory()).getChainingStrategy();
    // 链化策略必须 HEAD_WITH_SOURCES，输出边是可链化的
    return targetChainingStrategy == ChainingStrategy.HEAD_WITH_SOURCES
            && isChainableInput(sourceOutEdge, streamGraph, false);
}



private static boolean isChainableInput(
        StreamEdge edge, StreamGraph streamGraph, boolean allowChainWithDefaultParallelism) {
    StreamNode upStreamVertex = streamGraph.getSourceVertex(edge);
    StreamNode downStreamVertex = streamGraph.getTargetVertex(edge);

    if (!(streamGraph.isChainingEnabled()
            // 上下游节点是否在同一个 slot 共享组
            && upStreamVertex.isSameSlotSharingGroup(downStreamVertex)
            // 操作符是否可以链化，主要做并行度检查
            && areOperatorsChainable(
                    upStreamVertex,
                    downStreamVertex,
                    streamGraph,
                    allowChainWithDefaultParallelism)
            // 分区器和交换模式是否支持链化
            && arePartitionerAndExchangeModeChainable(
                    edge.getPartitioner(), edge.getExchangeMode(), streamGraph.isDynamic()))) {

        return false;
    }

    // check that we do not have a union operation, because unions currently only work
    // through the network/byte-channel stack.
    // we check that by testing that each "type" (which means input position) is used only once
    // 检查是否为 Union 操作，Union 操作不能链化
    for (StreamEdge inEdge : downStreamVertex.getInEdges()) {
        if (inEdge != edge && inEdge.getTypeNumber() == edge.getTypeNumber()) {
            return false;
        }
    }
    return true;
}
```

Source 的链化条件主要就是这些，我们结合一些例子来看一下。

```
Source(并行度=4) -> Map(并行度=4) -> Filter(并行度=4)

Source -> Map 边：
1. isChainingEnabled() = true
2. isSameSlotSharingGroup() = true (都在默认组)
3. areOperatorsChainable() = true (Source可链化，Map是HEAD_WITH_SOURCES)
4. arePartitionerAndExchangeModeChainable() = true (ForwardPartitioner)
5. Union检查通过
结果：可链化

Map -> Filter 边：
1. isChainingEnabled() = true
2. isSameSlotSharingGroup() = true
3. areOperatorsChainable() = true (Map和Filter都是ALWAYS)
4. arePartitionerAndExchangeModeChainable() = true (ForwardPartitioner)
5. Union检查通过
结果：可链化

最终：Source -> Map -> Filter 三者链化到一个JobVertex中


Source(并行度=2) -> Map(并行度=4)  // 并行度不匹配

Source -> Map 边：
1. isChainingEnabled() = true
2. isSameSlotSharingGroup() = true
3. areOperatorsChainable() = false (并行度不匹配)
结果：不可链化，需要网络传输



Source1 --\
          Union -> Map
Source2 --/

Source1 -> Union 边：
虽然满足前4个条件，但Union节点有两个输入边，typeNumber相同
Union检查失败，不可链化
```

得到了所有入口之后，就可以进行后续节点的链化操作了，它的逻辑在 createChain 方法中。这里主要是一个递归过程，先将节点的输出边分为可链化和不可链化两个 list，之后对可链化的边进行递归调用链化。对不可链化的边，需要创建出新的链。由于篇幅原因，这里只贴一部分核心的代码

```java
public static List<StreamEdge> createChain(
        final Integer currentNodeId,
        final int chainIndex,
        final OperatorChainInfo chainInfo,
        final Map<Integer, OperatorChainInfo> chainEntryPoints,
        final boolean canCreateNewChain,
        final Executor serializationExecutor,
        final JobVertexBuildContext jobVertexBuildContext,
        final @Nullable Consumer<Integer> visitedStreamNodeConsumer) {

    ......

        // 拆分可链化边和不可链化边
        for (StreamEdge outEdge : currentNode.getOutEdges()) {
            if (isChainable(outEdge, streamGraph)) {
                chainableOutputs.add(outEdge);
            } else {
                nonChainableOutputs.add(outEdge);
            }
        }

        // 处理可链化边
        for (StreamEdge chainable : chainableOutputs) {
            StreamNode targetNode = streamGraph.getStreamNode(chainable.getTargetId());
            Attribute targetNodeAttribute = targetNode.getAttribute();
            if (isNoOutputUntilEndOfInput) {
                if (targetNodeAttribute != null) {
                    targetNodeAttribute.setNoOutputUntilEndOfInput(true);
                }
            }
            transitiveOutEdges.addAll(
                    createChain(
                            chainable.getTargetId(),
                            chainIndex + 1,
                            chainInfo,
                            chainEntryPoints,
                            canCreateNewChain,
                            serializationExecutor,
                            jobVertexBuildContext,
                            visitedStreamNodeConsumer));
            // Mark upstream nodes in the same chain as outputBlocking
            if (targetNodeAttribute != null
                    && targetNodeAttribute.isNoOutputUntilEndOfInput()) {
                currentNodeAttribute.setNoOutputUntilEndOfInput(true);
            }
        }

        // 处理不可链化边
        for (StreamEdge nonChainable : nonChainableOutputs) {
            transitiveOutEdges.add(nonChainable);
            // Used to control whether a new chain can be created, this value is true in the
            // full graph generation algorithm and false in the progressive generation
            // algorithm. In the future, this variable can be a boolean type function to adapt
            // to more adaptive scenarios.
            if (canCreateNewChain) {
                createChain(
                        nonChainable.getTargetId(),
                        1, // operators start at position 1 because 0 is for chained source
                        // inputs
                        chainEntryPoints.computeIfAbsent(
                                nonChainable.getTargetId(),
                                (k) -> chainInfo.newChain(nonChainable.getTargetId())),
                        chainEntryPoints,
                        canCreateNewChain,
                        serializationExecutor,
                        jobVertexBuildContext,
                        visitedStreamNodeConsumer);
            }
        }

        // 创建 JobVertex
        StreamConfig config;
        if (currentNodeId.equals(startNodeId)) {
            JobVertex jobVertex = jobVertexBuildContext.getJobVertex(startNodeId);
            if (jobVertex == null) {
                jobVertex =
                        createJobVertex(
                                chainInfo, serializationExecutor, jobVertexBuildContext);
            }
            config = new StreamConfig(jobVertex.getConfiguration());
        } else {
            config = new StreamConfig(new Configuration());
        }

        // 判断是否为起始节点，如果不是，将对应的配置信息存到链化起始节点的 key 中
        if (currentNodeId.equals(startNodeId)) {
            chainInfo.setTransitiveOutEdges(transitiveOutEdges);
            jobVertexBuildContext.addChainInfo(startNodeId, chainInfo);

            config.setChainStart();
            config.setChainIndex(chainIndex);
            config.setOperatorName(streamGraph.getStreamNode(currentNodeId).getOperatorName());
            config.setTransitiveChainedTaskConfigs(
                    jobVertexBuildContext.getChainedConfigs().get(startNodeId));
        } else {
            config.setChainIndex(chainIndex);
            StreamNode node = streamGraph.getStreamNode(currentNodeId);
            config.setOperatorName(node.getOperatorName());
            jobVertexBuildContext
                    .getOrCreateChainedConfig(startNodeId)
                    .put(currentNodeId, config);
        }

    ......
}
```

是否可链化依赖于 isChainable 方法的结果。它主要判断了下游的输入边数量是否为1，然后调用了 isChainableInput，这个方法我们刚刚已经看过了。

```java
public static boolean isChainable(StreamEdge edge, StreamGraph streamGraph) {
    return isChainable(edge, streamGraph, false);
}

public static boolean isChainable(
        StreamEdge edge, StreamGraph streamGraph, boolean allowChainWithDefaultParallelism) {
    StreamNode downStreamVertex = streamGraph.getTargetVertex(edge);

    return downStreamVertex.getInEdges().size() == 1
            && isChainableInput(edge, streamGraph, allowChainWithDefaultParallelism);
}
```

### 总结

本文我们主要介绍了生成 JobGraph 的相关代码。首先了解了 JobGraph 中的节点和边对应的类，以及它们和 StreamGraph 中的类的映射关系。然后又看了生成 JobGraph 的核心代码，其中重点学习了链化相关的代码。

最后补充一个生成 JobGraph 的调用链路，感兴趣的同学可以看下。

```java
clusterClient.submitJob() → MiniCluster.submitJob() → Dispatcher.submitJob() → JobMasterServiceLeadershipRunnerFactory → DefaultJobMasterServiceFactory → JobMaster → DefaultSchedulerFactory.createInstance()→ StreamGraph.getJobGraph()
```
