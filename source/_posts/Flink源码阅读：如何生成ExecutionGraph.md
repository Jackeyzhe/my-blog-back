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

- **IntermediateResultPartition:** IntermediateResultPartition 是每个 ExecutionJobVertex 的输出。

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
