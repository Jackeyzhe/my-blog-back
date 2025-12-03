---
title: Flink源码阅读：状态管理
date: 2025-12-03 11:11:28
tags: Flink
---

在 [Flink学习笔记：状态类型和应用](https://jackeyzhe.github.io/2025/08/04/Flink%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9A%E7%8A%B6%E6%80%81%E7%B1%BB%E5%9E%8B%E5%92%8C%E5%BA%94%E7%94%A8/) 一文中，我们介绍了 Flink 状态的分类和应用。今天从源码层面再看一下 Flink 是如何管理状态的。
