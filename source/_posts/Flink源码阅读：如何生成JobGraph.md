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

JobVertex包含以下成员变量

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
