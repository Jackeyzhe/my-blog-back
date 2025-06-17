---
title: Flink学习笔记：时间、Watermark与窗口
date: 2025-06-17 00:26:16
tags: Flink
---

在前文中，我学习 Flink 的整体架构，接下来的几篇文章，我将重点学习一下 Flink 的几个核心概念。包括时间属性、Watermark、窗口、状态以及容错机制。今天就来学习前三个核心概念。<!-- more -->

### 时间属性

首先来学习 Flink 的时间属性，作为流处理引擎，时间是实时数据处理的重要依赖，特别是在做时序分析或者特定时间段数据处理时，时间的概念更显得尤为重要。

Flink 中支持三种时间属性，分别是：

- EventTime：事件时间，即为事件产生的时间。

- ProcessTime：处理时间，Flink 算子处理事件的时间。

- Ingestion Time：摄入时间，Flink 读取事件的时间。

这样描述可能比较抽象，我们通过一张图来看一下。

![FlinkTime](https://res.cloudinary.com/dxydgihag/image/upload/v1750094745/Blog/flink/1/flink_time.png)

从上图中可以看出，在时间产生/存储时，记录一个设备时间，就是 Event Time。当 Flink 的 DataSource 读取到事件时，这时再记录一个时间，这就是 Ingestion Time。在 Flink 程序中，每个算子处理事件时，又会记录一个时间，这个时间就是 Process Time。
