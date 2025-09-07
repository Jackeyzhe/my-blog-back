---
title: Flink学习笔记：反压
date: 2025-08-26 23:13:46
tags: Flink
---

今天来聊在 Flink 运维过程中比较常见的一个问题：反压。<!-- more -->

### 什么是反压

反压是流式系统中关于数据处理能力的动态反馈机制，并且是从下游到上游的反馈，一般发生在上游节点的生产速度大于下游节点的消费速度的情况。

### 数据如何传输

在了解反压的细节之前，首先要知道 Flink 中数据是如何传输的。在 Flink 中，两个算子之间的关系分为三种：

1. 部署在同一个 TaskManager 上，且属于同一算子链。

2. 部署在同一个 TaskManager 上，但不是同一个算子链。

3. 部署在不同的 TaskManager 上。

三种不同的关系，对应的算子间的数据传输方式也不同。先说第一种。

#### 同一线程数据传输

同一线程中的两个算子共享内存，因此数据传输非常简单，上游产出好数据后，直接调用下游的 processElement 方法即可。

#### 本地线程数据传输

对于第二种关系，两个算子不在同一线程，但是部署在同一个 TaskManager 上，也就是算子之间的数据传输是跨线程的。我们通过一个图来解释。

![本地线程传输](https://res.cloudinary.com/dxydgihag/image/upload/v1757168841/Blog/flink/7/Flink%E9%80%9A%E4%BF%A1.png)

图中，Flat Map Task 是上游算子，sum 是下游的算子。它们共享一块 Buffer 内存。当 Buffer 中没有数据可以消费时，sum 所在的线程会阻塞（步骤1）。随着数据的流入，Flat Map Task 会将处理好的数据写入到 ResultSubpartition（步骤2），然后 flush 到 Buffer 中（步骤3）。此时会唤醒 sum 所在的线程（步骤4），它就可以从 Buffer 中读取数据了（步骤5）。

#### 远程数据传输

第三种跨 TaskManager 的数据传输，与第二种类似，不过也有些区别。我们还是通过一张图来解释。

![远程数据传输](https://res.cloudinary.com/dxydgihag/image/upload/v1757170148/Blog/flink/7/Flink%E8%BF%9C%E7%A8%8B%E6%95%B0%E6%8D%AE%E4%BC%A0%E8%BE%93.png)

从图中可以看到，当 sum 所在线程没有 Buffer 可以消费时，会通过 PartitionRequestClient 向 Flat Map Task 所在的进程发送请求。Flat Map Task 所在进程接收到请求后，会读取 Buffer 中的数据并返回。

### Flink 的反压

了解了 Flink 的数据传输方式之后，我们再来看下 Flink 是如何感知反压的。

![Flink反压](https://res.cloudinary.com/dxydgihag/image/upload/v1757170638/Blog/flink/7/Flink%E5%8F%8D%E5%8E%8B%E8%BF%87%E7%A8%8B.png)

上图是一个数据传输的简图。当 Task1 有 Buffer 空间时，记录 A 被序列化并写入 LocalBufferPool 中，接着发送到 Task2 的 LocalBufferPool 中，Task2 读取并反序列化后交由程序处理。

这里我们也分两个场景讨论。

#### 本地传输

Task1 和 Task2 在同一个 TaskManager 节点，Task1 和 Task2 共用 Buffer，一旦 Task2 消费了 Buffer，该 Buffer 就会被回收。如果 Task2 的处理速度比 Task1 慢，那么 Buffer 的回收速度就赶不上 Task1 取 Buffer 的速度，这样会导致无 Buffer 可用，最终 Task1 就会降速。

#### 远程传输

Task1 和 Task2 运行在不同的 TaskManager 上，那 Buffer 会发送到网络后，等接收端消费完再回收。在发送端，会通过 Netty 水位机制来保证不往网络中写太多数据，如果网络中的数据超过了高水位值，就会等其下降到低水位值以下才会继续写数据。如果网络有堆积，发送端就会暂停发送，Buffer 也不会被回收，这就会阻塞 writer 往 ResultSubPartition 中写数据。

### 反压监控

在 Flink Web UI 中，可以找到反压的监控

![反压监控](https://res.cloudinary.com/dxydgihag/image/upload/v1757254334/Blog/flink/7/back_pressure_subtasks.png)

它有三种状态：

- **OK**: 0% <= 反压比例 <= 10%，此时一般不用处理。
- **LOW**: 10% < 反压比例 <= 50%，这种状态需要关注。
- **HIGH**: 50% < 反压比例 <= 100%，已经反压，需要赶快处理。

### 总结

今天我们聊了什么是反压，以及 Flink 中数据传输方法和 Flink 任务是如何感知反压的。Flink 的传输方式分为三种，分别是同线程传输、本地跨线程传输以及远程传输。Flink 任务在感知反压时也分别针对本地传输和远程传输做了讨论。
