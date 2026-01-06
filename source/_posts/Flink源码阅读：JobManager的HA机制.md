---
title: Flink源码阅读：JobManager的HA机制
date: 2026-01-06 22:15:45
tags: Flink
---

JobManager 在 Flink 集群中发挥着重要的作用，包括任务调度和资源管理等工作。如果 JobManager 宕机，那么整个集群的任务都将失败。为了解决 JobManager 的单点问题，Flink 也设计了 HA 机制来保障整个集群的稳定性。<!-- more -->

### 基本概念
