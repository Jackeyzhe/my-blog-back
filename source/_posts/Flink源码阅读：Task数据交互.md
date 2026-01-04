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

在 `StreamMap.processElement` 方法中，调用完 map 方法之后，就会调用 `output.collect` 方法将数据输出，这里的 output 就是 RecordWriterOutput。在 RecordWriterOutput 中，会调用 RecordWriter 的 emit 方法。

```java
private <X> void pushToRecordWriter(StreamRecord<X> record) {
    serializationDelegate.setInstance(record);

    try {
        recordWriter.emit(serializationDelegate);
    } catch (IOException e) {
        throw new UncheckedIOException(e.getMessage(), e);
    }
}
```

这里的 serializationDelegate 是用来对 record 进行序列化的。RecordWriter 有两个实现类，一个是 ChannelSelectorRecordWriter，另一个是 BroadcastRecordWriter。ChannelSelectorRecordWriter 需要先调用 ChannelSelector 选择对应的 subparition，然后进行写入。BroadcastRecordWriter 则是写到所有的 subparition。

接下来就是调用 `BufferWritingResultPartition.emitRecord` 来写入数据。

```java
public void emitRecord(ByteBuffer record, int targetSubpartition) throws IOException {
    totalWrittenBytes += record.remaining();

    BufferBuilder buffer = appendUnicastDataForNewRecord(record, targetSubpartition);

    while (record.hasRemaining()) {
        // full buffer, partial record
        finishUnicastBufferBuilder(targetSubpartition);
        buffer = appendUnicastDataForRecordContinuation(record, targetSubpartition);
    }

    if (buffer.isFull()) {
        // full buffer, full record
        finishUnicastBufferBuilder(targetSubpartition);
    }

    // partial buffer, full record
}
```

这里把 record 写入到 buffer 中，如果 buffer 不够，则会从 LocalBufferPool 中申请新的 buffer，申请到之后就会继续写入。下面是具体的申请过程。

```java
private MemorySegment requestMemorySegment(int targetChannel) {
    MemorySegment segment = null;
    synchronized (availableMemorySegments) {
        checkDestroyed();

        if (!availableMemorySegments.isEmpty()) {
            segment = availableMemorySegments.poll();
        } else if (isRequestedSizeReached()) {
            // Only when the buffer request reaches the upper limit(i.e. current pool size),
            // requests an overdraft buffer.
            segment = requestOverdraftMemorySegmentFromGlobal();
        }

        if (segment == null) {
            return null;
        }

        if (targetChannel != UNKNOWN_CHANNEL) {
            if (++subpartitionBuffersCount[targetChannel] == maxBuffersPerChannel) {
                unavailableSubpartitionsCount++;
            }
        }

        checkAndUpdateAvailability();
    }
    return segment;
}
```

如果有可用内存，就直接从队列中出队。如果达到了本地 BufferPool 的上限，就从全局的 NetworkBufferPool 中申请，申请不到就会阻塞写入过程，等待申请。最后还会检查并更新可用内存状态。

### 数据输入

看完了数据输出的过程之后，我们再来看一下数据输入的过程。首先还是了解几个基本概念。

#### 基本概念

- InputGate：

- InputChannel

#### 执行流程

![RecordInput](https://res.cloudinary.com/dxydgihag/image/upload/v1767451781/Blog/flink/18/RecordInput.png)
