---
title: Flink源码阅读：如何生成ExecutionGraph
date: 2025-11-08 23:23:57
tags: Flink
---

今天我们一起来了解 Flink 最后一种执行图，ExecutionGraph 的执行过程。<!-- more -->

### 基本概念

在阅读源码之前，我们先来了解一下 ExecutionGraph 中的一些基本概念。

- **ExecutionJobVertex:**  ExecutionJobVertex 是 ExecutionGraph 中的节点，对应的是 JobGraph 中的 JobVertex。

- **ExecutionVertex:** 每个 ExecutionJobVertex 都包含了一组 ExecutionVertex，ExecutionVertex 的数量就是节点对应的并行度。

- **IntermediateResult:** IntermediateResult 表示节点的输出结果，与之对应的是 JobGraph 中的 IntermediateDataSet。

- **IntermediateResultPartition:** IntermediateResultPartition 是每个 ExecutionVertex 的输出。

- **EdgeManager:** EdgeManager 主要负责存储 ExecutionGraph 中所有之间的连接，包括其并行度。

- **Execution:** Execution 可以认为是一次实际的运行尝试。每次执行时，Flink 都会将ExecutionVertex 封装成一个 Execution，并通过一个 ExecutionAttemptID 来做唯一标识。

### ExecutionGraph 生成过程

了解了这些基本概念之后，我们一起来看一下 ExecutionGraph 的具体生成过程。生成 ExecutionGraph 的代码入口是 DefaultExecutionGraphBuilder.build 方法。

首先是获取一些基本信息，包括 jobInformation、jobStatusChangedListeners 等。

接下来就是创建一个 DefaultExecutionGraph 和生成执行计划。

```java
// create a new execution graph, if none exists so far
final DefaultExecutionGraph executionGraph =
        new DefaultExecutionGraph(
                jobInformation,
                futureExecutor,
                ioExecutor,
                rpcTimeout,
                executionHistorySizeLimit,
                classLoader,
                blobWriter,
                partitionGroupReleaseStrategyFactory,
                shuffleMaster,
                partitionTracker,
                executionDeploymentListener,
                executionStateUpdateListener,
                initializationTimestamp,
                vertexAttemptNumberStore,
                vertexParallelismStore,
                isDynamicGraph,
                executionJobVertexFactory,
                jobGraph.getJobStatusHooks(),
                markPartitionFinishedStrategy,
                taskDeploymentDescriptorFactory,
                jobStatusChangedListeners,
                executionPlanSchedulingContext);


try {
    executionGraph.setPlan(JsonPlanGenerator.generatePlan(jobGraph));
} catch (Throwable t) {
    log.warn("Cannot create plan for job", t);
    // give the graph an empty plan
    executionGraph.setPlan(new JobPlanInfo.Plan("", "", "", new ArrayList<>()));
}
```

下面就是两个比较核心的方法 getVerticesSortedTopologicallyFromSources 和 attachJobGraph。

```java
// topologically sort the job vertices and attach the graph to the existing one
List<JobVertex> sortedTopology = jobGraph.getVerticesSortedTopologicallyFromSources();

executionGraph.attachJobGraph(sortedTopology, jobManagerJobMetricGroup);
```

这两个方法是先将 JobVertex 进行排序，然后构建 ExecutionGraph 的拓扑图。

#### getVerticesSortedTopologicallyFromSources

```java
public List<JobVertex> getVerticesSortedTopologicallyFromSources()
        throws InvalidProgramException {
    // early out on empty lists
    if (this.taskVertices.isEmpty()) {
        return Collections.emptyList();
    }

    List<JobVertex> sorted = new ArrayList<JobVertex>(this.taskVertices.size());
    Set<JobVertex> remaining = new LinkedHashSet<JobVertex>(this.taskVertices.values());

    // start by finding the vertices with no input edges
    // and the ones with disconnected inputs (that refer to some standalone data set)
    {
        Iterator<JobVertex> iter = remaining.iterator();
        while (iter.hasNext()) {
            JobVertex vertex = iter.next();

            if (vertex.isInputVertex()) {
                sorted.add(vertex);
                iter.remove();
            }
        }
    }

    int startNodePos = 0;

    // traverse from the nodes that were added until we found all elements
    while (!remaining.isEmpty()) {

        // first check if we have more candidates to start traversing from. if not, then the
        // graph is cyclic, which is not permitted
        if (startNodePos >= sorted.size()) {
            throw new InvalidProgramException("The job graph is cyclic.");
        }

        JobVertex current = sorted.get(startNodePos++);
        addNodesThatHaveNoNewPredecessors(current, sorted, remaining);
    }

    return sorted;
}
```

这段代码是将所有的节点进行排序，先将所有的 Source 节点筛选出来，然后再将剩余节点假如列表。这样就能构建出最终的拓扑图。

#### attachJobGraph

```java
@Override
public void attachJobGraph(
        List<JobVertex> verticesToAttach, JobManagerJobMetricGroup jobManagerJobMetricGroup)
        throws JobException {

    assertRunningInJobMasterMainThread();

    LOG.debug(
            "Attaching {} topologically sorted vertices to existing job graph with {} "
                    + "vertices and {} intermediate results.",
            verticesToAttach.size(),
            tasks.size(),
            intermediateResults.size());

    attachJobVertices(verticesToAttach, jobManagerJobMetricGroup);
    if (!isDynamic) {
        initializeJobVertices(verticesToAttach);
    }

    // the topology assigning should happen before notifying new vertices to failoverStrategy
    executionTopology = DefaultExecutionTopology.fromExecutionGraph(this);

    partitionGroupReleaseStrategy =
            partitionGroupReleaseStrategyFactory.createInstance(getSchedulingTopology());
}
```

attachJobGraph 方法主要包含两步逻辑，第一步是调用 attachJobVertices 方法创建 ExecutionJobVertex 实例，第二步是调用 fromExecutionGraph 创建一些其他的核心对象。

**attachJobVertices**

attachJobVertices 方法中就是遍历所有的 JobVertex，然后利用 JobVertex 生成 ExecutionJobVertex。

```java
/** Attach job vertices without initializing them. */
private void attachJobVertices(
        List<JobVertex> topologicallySorted, JobManagerJobMetricGroup jobManagerJobMetricGroup)
        throws JobException {
    for (JobVertex jobVertex : topologicallySorted) {

        if (jobVertex.isInputVertex() && !jobVertex.isStoppable()) {
            this.isStoppable = false;
        }

        VertexParallelismInformation parallelismInfo =
                parallelismStore.getParallelismInfo(jobVertex.getID());

        // create the execution job vertex and attach it to the graph
        ExecutionJobVertex ejv =
                executionJobVertexFactory.createExecutionJobVertex(
                        this,
                        jobVertex,
                        parallelismInfo,
                        coordinatorStore,
                        jobManagerJobMetricGroup);

        ExecutionJobVertex previousTask = this.tasks.putIfAbsent(jobVertex.getID(), ejv);
        if (previousTask != null) {
            throw new JobException(
                    String.format(
                            "Encountered two job vertices with ID %s : previous=[%s] / new=[%s]",
                            jobVertex.getID(), ejv, previousTask));
        }

        this.verticesInCreationOrder.add(ejv);
        this.numJobVerticesTotal++;
    }
}
```

**initializeJobVertices**

在 DefaultExecutionGraph.initializeJobVertices 中是遍历了刚刚排好序的 JobVertex，获取了 ExecutionJobVertex 之后调用了 ExecutionGraph.initializeJobVertex 方法。

我们直接来看 ExecutionGraph.initializeJobVertex 的逻辑。

```java
default void initializeJobVertex(ExecutionJobVertex ejv, long createTimestamp)
        throws JobException {
    initializeJobVertex(
            ejv,
            createTimestamp,
            VertexInputInfoComputationUtils.computeVertexInputInfos(
                    ejv, getAllIntermediateResults()::get));
}
```

这里先是调用了 VertexInputInfoComputationUtils.computeVertexInputInfos 方法，生成了 Map<IntermediateDataSetID, JobVertexInputInfo> jobVertexInputInfos。它表示的是每个 ExecutionVertex 消费上游 IntermediateResultPartition 的范围。

这里有两种模式，分别是 POINTWISE （点对点）和 ALL_TO_ALL（全对全）

在 POINTWISE 模式中，会按照尽量均匀分布的方式处理。

- 例如上游并发度是4，下游并发度是2时，那么前两个 IntermediateResultPartition 就会被第一个 ExecutionVertex 消费，后两个 IntermediateResultPartition 就会被第二个 ExecutionVertex 消费。

- 如果上游并发度是2，下游是3时，那么下游前两个 IntermediateResultPartition 会被第一个 ExecutionVertex 消费，第三个 IntermediateResultPartition 则会被第二个 ExecutionVertex 消费。

```java
public static JobVertexInputInfo computeVertexInputInfoForPointwise(
        int sourceCount,
        int targetCount,
        Function<Integer, Integer> numOfSubpartitionsRetriever,
        boolean isDynamicGraph) {

    final List<ExecutionVertexInputInfo> executionVertexInputInfos = new ArrayList<>();

    if (sourceCount >= targetCount) {
        for (int index = 0; index < targetCount; index++) {

            int start = index * sourceCount / targetCount;
            int end = (index + 1) * sourceCount / targetCount;

            IndexRange partitionRange = new IndexRange(start, end - 1);
            IndexRange subpartitionRange =
                    computeConsumedSubpartitionRange(
                            index,
                            1,
                            () -> numOfSubpartitionsRetriever.apply(start),
                            isDynamicGraph,
                            false,
                            false);
            executionVertexInputInfos.add(
                    new ExecutionVertexInputInfo(index, partitionRange, subpartitionRange));
        }
    } else {
        for (int partitionNum = 0; partitionNum < sourceCount; partitionNum++) {

            int start = (partitionNum * targetCount + sourceCount - 1) / sourceCount;
            int end = ((partitionNum + 1) * targetCount + sourceCount - 1) / sourceCount;
            int numConsumers = end - start;

            IndexRange partitionRange = new IndexRange(partitionNum, partitionNum);
            // Variable used in lambda expression should be final or effectively final
            final int finalPartitionNum = partitionNum;
            for (int i = start; i < end; i++) {
                IndexRange subpartitionRange =
                        computeConsumedSubpartitionRange(
                                i,
                                numConsumers,
                                () -> numOfSubpartitionsRetriever.apply(finalPartitionNum),
                                isDynamicGraph,
                                false,
                                false);
                executionVertexInputInfos.add(
                        new ExecutionVertexInputInfo(i, partitionRange, subpartitionRange));
            }
        }
    }
    return new JobVertexInputInfo(executionVertexInputInfos);
}
```

在 ALL_TO_ALL 模式中，每个下游都会消费所有上游的数据。

```java
public static JobVertexInputInfo computeVertexInputInfoForAllToAll(
        int sourceCount,
        int targetCount,
        Function<Integer, Integer> numOfSubpartitionsRetriever,
        boolean isDynamicGraph,
        boolean isBroadcast,
        boolean isSingleSubpartitionContainsAllData) {
    final List<ExecutionVertexInputInfo> executionVertexInputInfos = new ArrayList<>();
    IndexRange partitionRange = new IndexRange(0, sourceCount - 1);
    for (int i = 0; i < targetCount; ++i) {
        IndexRange subpartitionRange =
                computeConsumedSubpartitionRange(
                        i,
                        targetCount,
                        () -> numOfSubpartitionsRetriever.apply(0),
                        isDynamicGraph,
                        isBroadcast,
                        isSingleSubpartitionContainsAllData);
        executionVertexInputInfos.add(
                new ExecutionVertexInputInfo(i, partitionRange, subpartitionRange));
    }
    return new JobVertexInputInfo(executionVertexInputInfos);
}
```

生成好了 jobVertexInputInfos 之后，我们再回到 DefaultExecutionGraph.initializeJobVertex 方法中。

```java
@Override
public void initializeJobVertex(
        ExecutionJobVertex ejv,
        long createTimestamp,
        Map<IntermediateDataSetID, JobVertexInputInfo> jobVertexInputInfos)
        throws JobException {

    checkNotNull(ejv);
    checkNotNull(jobVertexInputInfos);

    jobVertexInputInfos.forEach(
            (resultId, info) ->
                    this.vertexInputInfoStore.put(ejv.getJobVertexId(), resultId, info));

    ejv.initialize(
            executionHistorySizeLimit,
            rpcTimeout,
            createTimestamp,
            this.initialAttemptCounts.getAttemptCounts(ejv.getJobVertexId()),
            executionPlanSchedulingContext);

    ejv.connectToPredecessors(this.intermediateResults);

    for (IntermediateResult res : ejv.getProducedDataSets()) {
        IntermediateResult previousDataSet =
                this.intermediateResults.putIfAbsent(res.getId(), res);
        if (previousDataSet != null) {
            throw new JobException(
                    String.format(
                            "Encountered two intermediate data set with ID %s : previous=[%s] / new=[%s]",
                            res.getId(), res, previousDataSet));
        }
    }

    registerExecutionVerticesAndResultPartitionsFor(ejv);

    // enrich network memory.
    SlotSharingGroup slotSharingGroup = ejv.getSlotSharingGroup();
    if (areJobVerticesAllInitialized(slotSharingGroup)) {
        SsgNetworkMemoryCalculationUtils.enrichNetworkMemory(
                slotSharingGroup, this::getJobVertex, shuffleMaster);
    }
}
```

首先来看 ExecutionJobVertex.initialize 方法。这个方法主要是生成 IntermediateResult 和 ExecutionVertex。

```java
protected void initialize(
        int executionHistorySizeLimit,
        Duration timeout,
        long createTimestamp,
        SubtaskAttemptNumberStore initialAttemptCounts,
        ExecutionPlanSchedulingContext executionPlanSchedulingContext)
        throws JobException {

    checkState(parallelismInfo.getParallelism() > 0);
    checkState(!isInitialized());

    this.taskVertices = new ExecutionVertex[parallelismInfo.getParallelism()];

    this.inputs = new ArrayList<>(jobVertex.getInputs().size());

    // create the intermediate results
    this.producedDataSets =
            new IntermediateResult[jobVertex.getNumberOfProducedIntermediateDataSets()];

    for (int i = 0; i < jobVertex.getProducedDataSets().size(); i++) {
        final IntermediateDataSet result = jobVertex.getProducedDataSets().get(i);

        this.producedDataSets[i] =
                new IntermediateResult(
                        result,
                        this,
                        this.parallelismInfo.getParallelism(),
                        result.getResultType(),
                        executionPlanSchedulingContext);
    }

    // create all task vertices
    for (int i = 0; i < this.parallelismInfo.getParallelism(); i++) {
        ExecutionVertex vertex =
                createExecutionVertex(
                        this,
                        i,
                        producedDataSets,
                        timeout,
                        createTimestamp,
                        executionHistorySizeLimit,
                        initialAttemptCounts.getAttemptCount(i));

        this.taskVertices[i] = vertex;
    }

    // sanity check for the double referencing between intermediate result partitions and
    // execution vertices
    for (IntermediateResult ir : this.producedDataSets) {
        if (ir.getNumberOfAssignedPartitions() != this.parallelismInfo.getParallelism()) {
            throw new RuntimeException(
                    "The intermediate result's partitions were not correctly assigned.");
        }
    }

    // set up the input splits, if the vertex has any
    try {
        @SuppressWarnings("unchecked")
        InputSplitSource<InputSplit> splitSource =
                (InputSplitSource<InputSplit>) jobVertex.getInputSplitSource();

        if (splitSource != null) {
            Thread currentThread = Thread.currentThread();
            ClassLoader oldContextClassLoader = currentThread.getContextClassLoader();
            currentThread.setContextClassLoader(graph.getUserClassLoader());
            try {
                inputSplits =
                        splitSource.createInputSplits(this.parallelismInfo.getParallelism());

                if (inputSplits != null) {
                    splitAssigner = splitSource.getInputSplitAssigner(inputSplits);
                }
            } finally {
                currentThread.setContextClassLoader(oldContextClassLoader);
            }
        } else {
            inputSplits = null;
        }
    } catch (Throwable t) {
        throw new JobException(
                "Creating the input splits caused an error: " + t.getMessage(), t);
    }
}
```

在创建 ExecutionVertex 时，会创建 IntermediateResultPartition 和 Execution，创建 Execution 时，会设置 attemptNumber，这个值默认是0，如果 ExecutionVertex 是重新调度的，那么 attemptNumber 会自增加1。

ExecutionJobVertex.connectToPredecessors 方法主要是生成 ExecutionVertex 与 IntermediateResultPartition 的关联关系。这里设置关联关系也分成了点对点和全对全两种模式处理，点对点模式需要计算 ExecutionVertex 对应的 IntermediateResultPartition index 的范围。两种模式最终都调用了 connectInternal 方法。

```java
/** Connect all execution vertices to all partitions. */
private static void connectInternal(
        List<ExecutionVertex> taskVertices,
        List<IntermediateResultPartition> partitions,
        ResultPartitionType resultPartitionType,
        EdgeManager edgeManager) {
    checkState(!taskVertices.isEmpty());
    checkState(!partitions.isEmpty());

    ConsumedPartitionGroup consumedPartitionGroup =
            createAndRegisterConsumedPartitionGroupToEdgeManager(
                    taskVertices.size(), partitions, resultPartitionType, edgeManager);
    for (ExecutionVertex ev : taskVertices) {
        ev.addConsumedPartitionGroup(consumedPartitionGroup);
    }

    List<ExecutionVertexID> consumerVertices =
            taskVertices.stream().map(ExecutionVertex::getID).collect(Collectors.toList());
    ConsumerVertexGroup consumerVertexGroup =
            ConsumerVertexGroup.fromMultipleVertices(consumerVertices, resultPartitionType);
    for (IntermediateResultPartition partition : partitions) {
        partition.addConsumers(consumerVertexGroup);
    }

    consumedPartitionGroup.setConsumerVertexGroup(consumerVertexGroup);
    consumerVertexGroup.setConsumedPartitionGroup(consumedPartitionGroup);
}
```

这个方法中 ev.addConsumedPartitionGroup(consumedPartitionGroup); 负责将 ExecutionVertex 到 IntermediateResultPartition 的关联关系保存在 EdgeManager.vertexConsumedPartitions 中。

而 partition.addConsumers(consumerVertexGroup); 则负责将 IntermediateResultPartition 到 ExecutionVertex 的关系保存在 EdgeManager.partitionConsumers 中。

### 总结

通过本文，我们了解了 Flink 是如何将 JobGraph 转换成 ExecutionGraph 的。其中涉及到的一些核心概念名称比较类似，建议认真学习和理解透彻之后再研究其生成方法和对应关系，也可以借助前文中 ExecutionGraph 示意图辅助学习。
