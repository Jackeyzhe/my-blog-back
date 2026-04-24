---
title: 从零学习Kafka：幂等与事务
date: 2026-04-23 13:45:29
tags: Kafka
---

前文我们聊了生产者压缩机制相关的知识点，压缩机制主要是为了节省资源并提高吞吐。本文我们再来看一下 Kafka 的可靠性机制。<!-- more -->

### 写在前面

在正式开始之前，先来回顾一下前面留的一个问题：Consumer 在解压数据时，需要关系 Producer 使用了哪种压缩算法吗？

答案是不需要，在 Kafka 消息批次的头信息中，包含有 Attribute 字段，这个字段中有几位专门用来标识这个批次所使用的压缩算法。Consumer 在拿到数据后，如果发现是压缩数据，会先使用对应的压缩算法解压，然后把解压后的数据交给业务代码。

### 可靠性保证

在实时计算领域，可靠性保证通常都有三种：

- At most once：最多一次，即消息可能丢失，但不会重复。

- At least once：最少一次，即消息不会丢失，但可能重复。

- Exactly once：精确一次，消息不会丢失，也不会重复。

Kafka 默认支持的可靠性保证是第二种——最少一次。其原理也很简单，就是 Producer 端在没有收到 Broker 的确认消息时，会进行重试。支持最多一次的方法也很简单，只需要禁用重试就可以了，这样 Producer 写入消息可能失败也可能成功，但不会重复。

在对于消息准确性要求高的场景中，最多一次和最少一次都不能满足要求，需要保证精确一次才可以。在 Flink 中是通过 Checkpoint 来保证精确一次的，而在 Kafka 中是通过幂等和事务来保证的。

### 幂等性

#### 什么是幂等性

我们先来看幂等性，首先解释一下什么是幂等性。在数学中的定义是，对于一个函数 f(x) 作用在任意一个元素上一次和多次，得到的结果是一样的。例如乘以 1 这个操作，一个数乘以一次 1 和乘以 n 次 1，得到的结果都相同，乘以 1 这个操作就是幂等的。而加 1 就不是。

在计算机领域，幂等性的定义是用户对同一个操作发起一次或多次请求，最终资源的状态是不变的。这就保证了我们可以放心的重试，不需要担心数据重复。

#### Kafka的幂等性

我们知道了幂等性的定义之后，重点来关注一下 Kafka 是如何开启幂等性的。默认情况下，Producer 不是幂等性的，我们可以通过设置一个参数来创建幂等性的 Producer。

```java
props.put("enable.idempotence", true);
// 或者
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG， true);
```

那么 Kafka 是如何实现幂等性的呢？答案是通过引入两个关键标识：Producer ID 和 Sequence Number。

- Producer ID：当一个启用了幂等性的 Producer 启动时，Broker 会为它分配一个唯一的 ID，这个 ID 在 Producer 生命周期内保持不变。

- Sequence Number：Producer 为发往每个分区的每条消息都附带一个从 0 开始的递增的序列号。

有了这两个标识之后，Broker 会在内存中维护一个 (Producer ID, Sequence Number) 组合的高水位线，即 last_sequence_number。在消息写入过程中可能会遇到以下三种情况：

1. 正常写入：收到的 Sequence Number 比 last_sequence_number 大 1，Broker 接收消息并更新 last_sequence_number。

2. 重复消息：如果收到的 Sequence Number 小于或等于 last_sequence_number，那么代表这个消息之前已经收到过了，Broker 会丢弃消息，但会返回给 Producer 写入成功。

3. 丢消息/乱序：如果 Sequence Number 比 last_sequence_number 大了不止 1，说明中间漏了消息，可能是迟到或者缺失，此时 Broker 会报错。

聊到这里，你是不是觉得启用了幂等性的 Producer 之后，貌似就能保证消息不重不丢了。前面我们提到了 Producer ID 在 Producer 生命周期内保持不变，如果 Producer 重启了，那么 Broker 会为它分配一个新的 Producer ID，这样就无法保证消息不重复了。幂等性的另一个局限是跨分区的原子性无法保证。针对这两个问题 Kafka 引入了事务。

### 事务

从 0.11 版本开始，Kafka 提供了对事务的支持。关于事务的定义我们就不过多介绍了，就是 ACID。在 Kafka 中就是解决跨分区的原子性和 Producer 重启可能导致的消息重复。

开启事务需要进行两项配置，同时要修改发送消息的代码。

- 开启幂等性，设置 enable.idempotence=true

- 设置 Producer 的 transaction.id，最好设置成一个有意义的名字

发送消息的代码需要增加事务相关的处理

```java
producer.initTransactions();
try { 
    producer.beginTransaction();
    producer.send(record1);
    producer.send(record2);
    producer.commitTransaction();
} catch (KafkaException e) { 
    producer.abortTransaction();
}
```

#### 事务流程

为了支持事务，Kafka 引入了一个核心组件事务协调器和一个内部 Topic `__transaction_state`。整个过程基本就是两阶段提交的过程。

##### 1. 初始化

Producer 启动时，会向 Broker 发送请求，确定属于自己的事务协调器，然后告诉事务协调器自己的 transaction.id。事务协调器收到请求之后，会记录 transaction.id 的 epoch，这是一个递增的批次号，有了这个批次号之后，旧的生产者就会被屏蔽。

##### 2. 开启事务

初始化完成之后，需要开启事务，即调用 `beginTransaction()` 方法。这是在告诉事务协调器，我要开始往 Topic1 的分区 0 发送消息了。事务协调器就会把这些信息记录在`__transaction_state` 这个 Topic 中。

##### 3. 发送消息

此时 Producer 开始发送消息，这些消息会写入 Broker 的磁盘，但它们会带有一个“事务进行中”的标识。

##### 4. 提交阶段

发送完消息后，生产者调用 `commitTransaction()` 方法来提交事务，这里有两个步骤。

1. 准备阶段：事务协调器将事务状态改为 **PrepareCommit** 并写入 `__transaction_state` 。

2. 事务协调器向所有的分区 Leader 发送 `WriteTxnMarker` 请求，各个分区 Leader 会写入一个控制消息（Control Marker），如果中途有失败，事务协调器会发送 Abort 标记。

##### 5. 完成

当所有的分区的 Marker 都写入完成时，事务协调器将事务状态改为 Complete，否则标记为 Abort。

至此，一次完整的事务消息发送的过程就结束了。

### 如何Exactly Once

前面的过程中，不管事务成功与否，消息都被写入到了 Broker 的磁盘中了，那如何保证 Exactly Once 呢？实际上在 Consumer 端还有一个参数控制：`isolation.level`，它用来控制隔离级别。这个参数取值有两个：

- read_uncommitted：这是默认的，读未提交。也就是说，不管消息是什么状态，只要写到了 Broker 的磁盘上，都能被 Consumer 读到。

- read_committed：这是读已提交，也是我们开启事务时需要的。它表明我们只能看到提交的事务消息。

这样幂等性+事务+隔离级别三者配合起来，Kafka 就能支持精确一次的可靠性保证了。

### 总结

我们来总结一下今天的内容，首先介绍了 Kafka 保证的可靠性，默认是最少一次，可以支持最多一次和精确一次。精确一次的支持依赖于幂等性、事务和隔离级别。我们从这三个方面分别介绍了在 Kafka 中是如何支持的。

最后我们留下一个问题。如果生产者已经发送了大量消息，但在最后提交之前突然宕机，事务协调器会如何处理这个未完成的事务呢？
