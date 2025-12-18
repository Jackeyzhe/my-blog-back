---
title: Flink源码阅读：Watermark机制
date: 2025-12-17 15:15:22
tags: Flink
---

前面我们已经梳理了 Flink 状态和 Checkpoint 相关的源码。从本文开始，我们再来关注另外几个核心概念，即时间、Watermark 和窗口。<!-- more -->

### 写在前面

在 Flink 中 Watermark 是用来解决数据乱序问题的，它也是窗口关闭的触发条件。对于 Watermark 的概念和用法还不熟悉的同学可以先阅读[Flink学习笔记：时间与Watermark](https://jackeyzhe.github.io/2025/06/30/Flink%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9A%E6%97%B6%E9%97%B4%E4%B8%8EWatermark/)一文。下面我们进入正题，开始梳理 Watermark 相关的源码。

### Watermark 定义

Watermark 的定义非常简单，它继承了 `StreamElement` 类，内部只有一个 timestamp 变量。

```java
@PublicEvolving
public class Watermark extends StreamElement {

    /** The watermark that signifies end-of-event-time. */
    public static final Watermark MAX_WATERMARK = new Watermark(Long.MAX_VALUE);

    /** The watermark that signifies is used before any actual watermark has been generated. */
    public static final Watermark UNINITIALIZED = new Watermark(Long.MIN_VALUE);

    // ------------------------------------------------------------------------

    /** The timestamp of the watermark in milliseconds. */
    protected final long timestamp;

    /** Creates a new watermark with the given timestamp in milliseconds. */
    public Watermark(long timestamp) {
        this.timestamp = timestamp;
    }

    /** Returns the timestamp associated with this {@link Watermark} in milliseconds. */
    public long getTimestamp() {
        return timestamp;
    }

    // ------------------------------------------------------------------------

    @Override
    public boolean equals(Object o) {
        return this == o
                || o != null
                        && o.getClass() == this.getClass()
                        && ((Watermark) o).timestamp == timestamp;
    }

    @Override
    public int hashCode() {
        return (int) (timestamp ^ (timestamp >>> 32));
    }

    @Override
    public String toString() {
        return "Watermark @ " + timestamp;
    }
}
```

### Watermark 处理过程

我们先来回顾一下 Watermark 的生成方法。

```java
SingleOutputStreamOperator<Event> withTimestampsAndWatermarks = source
        .assignTimestampsAndWatermarks(
                WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(20))
        );
```

#### 初始化

在定义 Watermark 的时候，我们调用 assignTimestampsAndWatermarks 方法。

```java
public SingleOutputStreamOperator<T> assignTimestampsAndWatermarks(
        WatermarkStrategy<T> watermarkStrategy) {
    final WatermarkStrategy<T> cleanedStrategy = clean(watermarkStrategy);
    // match parallelism to input, to have a 1:1 source -> timestamps/watermarks relationship
    // and chain
    final int inputParallelism = getTransformation().getParallelism();
    final TimestampsAndWatermarksTransformation<T> transformation =
            new TimestampsAndWatermarksTransformation<>(
                    "Timestamps/Watermarks",
                    inputParallelism,
                    getTransformation(),
                    cleanedStrategy,
                    false);
    getExecutionEnvironment().addOperator(transformation);
    return new SingleOutputStreamOperator<>(getExecutionEnvironment(), transformation);
}
```

这个方法接收了一个 WatermarkStrategy 参数，把它封装到 TimestampsAndWatermarksTransformation 中之后，就添加到 transformations 列表中了。在生成 StreamGraph 的过程中，会调用每个 transformation 的 transform 方法。

![transform](https://res.cloudinary.com/dxydgihag/image/upload/v1766047572/Blog/flink/15/watermarkTranslate.png)

通过这个调用链路，创建出了 TimestampsAndWatermarksOperatorFactory，在初始化 StreamTask 时，会调用 TimestampsAndWatermarksOperatorFactory.createStreamOperator 方法来创建 TimestampsAndWatermarksOperator，并调用它的 open 方法。

在这个 open 方法中，主要是生成 timestampAssigner 和 watermarkGenerator。timestampAssigner 是用于提取时间戳，watermarkGenerator 是用于生成 Watermark。

生成完成之后注册了一个定时器，到指定时间后会调用 onProcessingTime 方法。

```java
public void onProcessingTime(long timestamp) throws Exception {
    watermarkGenerator.onPeriodicEmit(wmOutput);

    final long now = getProcessingTimeService().getCurrentProcessingTime();
    getProcessingTimeService().registerTimer(now + watermarkInterval, this);
}
```

这个方法的逻辑也很简单，先发送创建并发送 Watermark，然后再注册一个定时器。

#### 发送 Watermark
