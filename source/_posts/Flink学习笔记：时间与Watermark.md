---
title: Flink学习笔记：时间与Watermark
date: 2025-06-30 22:58:16
tags: Flink
---

在前文中，我学习 Flink 的整体架构，接下来的几篇文章，我将重点学习一下 Flink 的几个核心概念。包括时间属性、Watermark、窗口、状态以及容错机制。今天就来学习时间属性和 Watermark。<!-- more -->

### 时间属性

首先来学习 Flink 的时间属性，作为流处理引擎，时间是实时数据处理的重要依赖，特别是在做时序分析或者特定时间段数据处理时，时间的概念更显得尤为重要。

Flink 中支持三种时间属性，分别是：

- EventTime：事件时间，即为事件产生的时间。

- ProcessTime：处理时间，Flink 算子处理事件的时间。

- IngestionTime：摄入时间，Flink 读取事件的时间。

这样描述可能比较抽象，我们通过一张图来看一下。

![FlinkTime](https://res.cloudinary.com/dxydgihag/image/upload/v1750094745/Blog/flink/1/flink_time.png)

从上图中可以看出，在时间产生/存储时，记录一个设备时间，就是 Event Time。当 Flink 的 DataSource 读取到事件时，这时再记录一个时间，这就是 Ingestion Time。在 Flink 程序中，每个算子处理事件时，又会记录一个时间，这个时间就是 Process Time。

### Watermark

介绍完了时间概念，再来看下 Watermark 的概念。它是 Flink 处理迟到事件的妙招。

Watermark 本身也属于一种特殊的事件，它由 Source 生成，同时携带由 Timestamp，并且会跟随正常的事件一起在 Flink 算子之间流转。Watermark 的作用是定义何时停止等待较早的事件。这么介绍可能比较抽象，下面我们通过一些具体的例子来进行更进一步的说明。

![watermark](https://res.cloudinary.com/dxydgihag/image/upload/v1751033706/Blog/flink/1/watermark.png)

上图代表的是一段乱序的事件数据流。假设我们定义 maxOutOfOrderness 为4，也就是容忍最大迟到时间为4（这里不带具体时间单位，可能是4秒也可能是4分钟）。当我们收到时间戳为7的事件时，就会生成一个时间为3的 Watermark。这代表着3之前的数据都已就绪。如果此时再有小于3的数据，我们认为它是迟到数据。

而对于迟到的数据，通常有三种处理方法：

- 重新开启已经关闭的窗口，重新计算并修正结果

- 将迟到事件使用旁路输出收集起来单独处理

- 将迟到事件视为错误消息丢弃

在 Flink 中 Watermark 本身是没有意义的，它的主要作用是作为窗口的触发条件。窗口可以认为是一个时间段，它有开始时间和结束时间。在窗口内可以计算一批事件的统计结果。关于窗口，我们后面再做详细介绍。

那么 Watermark 是如何触发窗口的呢？答案是必须要满足以下两个条件：

1. Watermark 的时间戳 >= 窗口的 end_time

2. 窗口中有数据

从概念上看还是比较抽象，我们还用上面的数据流作为例子，Watermark 设置为最大时间减 4，假设我们设置10秒一个窗口。关键代码如下：

```java
***
SingleOutputStreamOperator<Event> withTimestampsAndWatermarks = source
                .assignTimestampsAndWatermarks(
                        WatermarkStrategy.forGenerator(ctx -> new CustomWatermarkGenerator())
                                .withTimestampAssigner(((event, l) -> event.timestamp))
                );

OutputTag<Event> lateTag = new OutputTag<Event>("late-tag") {};

SingleOutputStreamOperator<String> windowResult = withTimestampsAndWatermarks
        .keyBy(event -> event.num)
        .window(TumblingEventTimeWindows.of(Time.seconds(10)))
        .sideOutputLateData(lateTag)
        .process(new ProcessWindowFunction<Event, String, Long, TimeWindow>() {
         @Override
         public void process(Long key, Context context, Iterable<Event> elements, Collector<String> out) {
             // 一些逻辑处理
             out.collect(result);
         }
});

// 处理迟到数据
DataStream<Event> lateStream = windowResult.getSideOutput(lateTag);
lateStream.process(new ProcessFunction<Event, String>() {
    @Override
    public void processElement(Event event, Context ctx, Collector<String> out) {
        out.collect("迟到事件: " + event);
    }
}).print();
***


***
@Override
public void onEvent(Event event, long l, WatermarkOutput watermarkOutput) {
    long eventTime = event.timestamp;
    // 使用CAS确保线程安全
    while (true) {
        long current = currentMaxTime.get();
        if (eventTime <= current) break;
        if (currentMaxTime.compareAndSet(current, eventTime)) break;
    }
}

@Override
public void onPeriodicEmit(WatermarkOutput watermarkOutput) {
    watermarkOutput.emitWatermark(new Watermark(currentMaxTime.get() - timeDiff));
}
***
```

[完整代码我放在 github 上了](https://github.com/Jackeyzhe/flink-training/blob/feature/wz-demo/common/src/main/java/org/apache/flink/training/examples/watermark/WatermarkDemo.java)

当我们输入测试数据时

```java
4,1750867204000
2,1750867202000
7,1750867207000
10,1750867210000
9,1750867209000
15,1750867215000
12,1750867212000
13,1750867213000
25,1750867225000
14,1750867214000
35,1750867235000
```

可以看到如下输出：

![print](https://res.cloudinary.com/dxydgihag/image/upload/v1751035105/Blog/flink/1/%E6%88%AA%E5%B1%8F2025-06-27_22.35.50.png)

通过输出的日志，我们可以看出，当watermark推进到大于等于时间窗口的结束时间时，窗口就会完成计算并关闭。而对于迟到的数据，我们可以通过侧输出流单独处理，也可以通过设置`allowedLateness`，使窗口重新打开。

### 生成 Watermark

了解了 Watermark 的原理之后，我们再来看一下如何生成 Watermark。在 Flink 中，需要使用 WatermarkStrategy 来定义如何生成时间戳和 watermark。WatermarkStrategy 继承了 TimestampAssignerSupplier 和 WatermarkGeneratorSupplier 两个接口，其中 TimestampAssignerSupplier 定义了抽取 EventTime 的方法，而 WatermarkGeneratorSupplier 则是定义了如何生成 Watermark 的方法。

#### Flink 内置的 Watermark 生成器

Flink 中内置了两个 watermark 生成器。分别是 AscendingTimestampsWatermarks 和 BoundedOutOfOrdernessWatermarks。

我们先来看 BoundedOutOfOrdernessWatermarks，它定义了一个 watermark 滞后于最大事件时间一个固定值的 watermark 生成器。在使用时，可以给定一个时间，这样 Flink 就会 根据最大的 eventTime 来周期性的生成 watermark，例如，我们前面定义的 watermark 滞后4秒，就可以写成：

```java
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(4));
```

AscendingTimestampsWatermarks 是单调递增时间分配器，也就是只处理有序的数据，它继承了 BoundedOutOfOrdernessWatermarks，并且最大容忍时间为0。在使用时，可以直接通过以下方法生成：

```java
WatermarkStrategy.forMonotonousTimestamps();
```

#### 自定义 WatermarkGenerator

除了上面两个内置的 WatermarkGenerator 外，我们还可以自定义，实现起来也比较简单。只需要实现 WatermarkGenerator 接口并重写 onEvent 和 onPeriodicEmit 两个方法即可。onEvent 是每个事件到来时调用一次，可以用来记录最大事件时间。onPeriodicEmit 则是周期性调用，可以生成 watermark。在前面的例子中，我使用的 CustomWatermarkGenerator 就是自定义的 watermark，对应的实现也在前文中贴了。

### 如何处理空闲数据源

最后，再补充一个与 watermark 相关的比较重要的特性。在 Flink 中，会有一些算子有多个输入源。这时，这个算子的 watermark 是以它收到的数据源中最小的 eventTime 来计算的。直接看官网的例子：

![parallel_streams_watermarks](https://res.cloudinary.com/dxydgihag/image/upload/v1752592909/Blog/flink/1/parallel_streams_watermarks.png)

那么这里就存在一个问题：如果一个输入源数据量很少，很久才发一条消息，而另一个数据源发了很多消息，那么就会在下游算子中积累很多消息等待处理，这对于整个系统的稳定性造成了很大的风险。

那这种情况有办法处理吗？答案是肯定的，Flink 提供了 withIdleness 方法，它可以用来检测空闲数据源，如果超过一定时间没有数据到来，Flink 认为这个数据源属于空闲数据源，这时就不会再阻塞下游算子触发窗口。达到定期处理数据的目的。

## 总结

今天我们先了解了 Flink 中时间的概念，EventTime 是事件产生的时间，通常由上游数据源生成，ProcessTime 是处理时间，通常由处理算子本身生成，IngestionTime是摄入时间，通常由 Flink 的 Source 生成。

接着我们由了解了 Flink 的 watermark，它是窗口触发的条件，在处理迟到数据时发挥着重要的作用。我们可以定义可以容忍的最大迟到时间，这样当遇到乱序数据时也可以得到正确的结果。
