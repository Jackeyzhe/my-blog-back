---
title: Flink源码阅读：Task数据交互
date: 2025-12-29 10:44:46
tags: Flink
---

经过前面的学习，Flink 的几个核心概念相关的源码实现我们已经了解了。本文我们来梳理 Task 的数据交互相关的源码。<!-- more -->

### 数据输出

话不多说，我们直接进入正题。首先来看 Task 的数据输出，在进入流程之前，我们先介绍几个基本概念。

#### 基本概念

- RecordWriterOutput：它是 Output 接口的一个具体实现类，底层使用 RecordWriter 来发送数据。

- RecordWriter：数据写入的执行者，负责将数据写到 ResultPartition。

- ResultPartition 和 ResultSubpartition：ResultPartition 是 ExecutionGraph 中一个节点的输出结果，下游的每个需要从当前 ResultPartition 消费数据的 Task 都会有一个 ResultSubpartition。

- ChannelSelector：用来决定一个 Record 要被写到哪个 Subpartition 中。

- LocalBufferPool：用来管理 Buffer 的缓冲池。在介绍反压的原理时，我们提到过。

对这些基本概念有了一定的了解之后，我们来看数据输出的具体流程。

#### 执行流程

我们以 map 为例，看一下数据的输出过程。

![DataOutput](https://res.cloudinary.com/dxydgihag/image/upload/v1767175113/Blog/flink/18/RecordOutput.png)
