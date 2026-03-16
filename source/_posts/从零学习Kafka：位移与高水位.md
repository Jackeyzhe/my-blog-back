---
title: 从零学习Kafka：位移与高水位
date: 2026-02-06 11:15:04
tags: Kafka
---

还记得上一篇文章最后的问题吗，什么是 LEO（Log End Offset）？它其实是 Kafka 位移相关的一个核心概念，本文我们就从位移开始，把相关的概念理清楚。<!-- more -->

### 什么是 Offset

我们从 Offset 说起，它是一个单调递增的 64 位整数，当 Producer 把数据写到 Partition 后，Kafka 先为消息分配一个编号，然后再写入到文件中。这个编号就是 Offset，它就像我们去银行办业务，每个人都会拿一个号，后面的人的号总是比前面的大 1，每个人的号都是唯一的。

### LEO 和 HW

了解了什么是 Offset 之后，我们再一起来认识今天的两位主角：LEO 和 HW。

![LEO](https://res.cloudinary.com/dxydgihag/image/upload/v1772012210/Blog/Kafka/4/Kafka_LEO.png)

LEO 是 Log End Offset 的缩写，也就是日志末端位移，它表示副本写入下一条消息的位移值。上图中一共有12条消息（位移值为 0 到 11），下一条消息的位移值就是 12，也就是当前副本的 LEO 为 12。

HW 是 High Watermark的缩写，也就是高水位。它用来定义消息的可见性，即哪些消息是可以被消费的。Offset 小于高水位的消息都是已提交消息，是可以被消费的。大于等于高水位值的消息都是未提交消息，这些消息不能够被下游消费。

一个分区的每个副本都会有一组 HW 和 LEO 值，分区的消息是否可以被消费取决于 Leader 副本的高水位值，也就是说，Leader 副本的高水位值就是分区高水位值。这里就会引出另一个问题，Follower 副本与 Leader 副本如何同步高水位和 LEO 值呢？

#### 同步流程

在了解高水位的同步流程之前，我们先看一下 LEO 和 HW 的存储。

![HW存储](https://res.cloudinary.com/dxydgihag/image/upload/v1772445493/Blog/Kafka/4/Kafka_HW.png)

在 Broker0 中，保存了某个分区 Leader 副本的 HW 和 LEO 值，以及这个分区所有的 Follower 副本的 LEO 值，这些 Follower 副本我们称之为远程副本，即 Remote Replica。
