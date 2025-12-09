---
title: Flink源码阅读：Checkpoint机制
date: 2025-12-09 15:14:59
tags: Flink
---

前文我们梳理了 Flink 状态管理相关的源码，我们知道，状态是要与 Checkpoint 配合使用的。因此，本文我们就一起来看一下 Checkpoint 相关的源码。<!-- more -->

### 说在前面
