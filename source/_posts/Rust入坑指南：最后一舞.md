---
title: Rust入坑指南：最后一舞
date: 2020-04-19 21:15:29
tags: Rust
---

Rust入坑指南系列我觉得应该告一段落了，最后来做一个总结吧。<!-- more -->

在我看来，Rust语言本身设计得算是非常好了。Ownership和borrow帮助我们保证了程序了安全性。同时也提供了Unsafe，给开发者更多玩一些骚操作的空间。唯一的缺点就是入门比较困难了吧，我现在的水平感觉自己也就是刚刚入门。而《Rust入坑指南》也是希望帮助更多想要学习Rust的同学快速入门。

这里简单回顾一下学习过程吧。

最开始接触一门语言一定绕不开Hello，World

Rust也是一样，所以我们的第一篇文章就是关于Rust的安装、Hello，World程序的。

[Rust入坑指南：坑主驾到](https://jackeyzhe.github.io/2019/09/21/Rust入坑指南：坑主驾到/)

接着就是介绍一些基础的语法、Rust的所有权、数据结构这些概念。关于这部分知识我用了四篇文章来做介绍。其中最重要的应该是Rust所有权了，这也是Rust语言的亮点之一。

[Rust入坑指南：常规套路](https://jackeyzhe.github.io/2019/10/08/Rust入坑指南：常规套路/)

[Rust入坑指南：核心概念](https://jackeyzhe.github.io/2019/10/13/Rust入坑指南：核心概念/)

[Rust入坑指南：千人千构](https://jackeyzhe.github.io/2019/10/27/Rust入坑指南：千人千构/)

[Rust入坑指南：鳞次栉比](https://jackeyzhe.github.io/2019/11/27/Rust入坑指南：鳞次栉比/)

接着呢，我们介绍了Package和Crate，用来帮助我们组织代码的。同时Crate也是为了让我们可以直接使用别人的代码，避免重复造轮子。

[Rust入坑指南：有条不紊](https://jackeyzhe.github.io/2019/11/03/Rust入坑指南：有条不紊/)

之后又是两个比较通用的概念，大多数编程语言时都要涉及到的：异常处理和泛型

[Rust入坑指南：亡羊补牢](https://jackeyzhe.github.io/2019/12/30/Rust入坑指南：亡羊补牢/)

[Rust入坑指南：海纳百川](https://jackeyzhe.github.io/2020/01/14/Rust入坑指南：海纳百川/)

如果你对代码的正确性不放心，那么一定要写下完备的单元测试，这是对自己的代码负责。

[Rust入坑指南：步步为营](https://jackeyzhe.github.io/2020/02/21/Rust入坑指南：步步为营/)

除了OwnerShip和borrow之外，Rust的另外两个比较核心的概念也需要了解，分别是生命周期和智能指针。这两篇文章可以帮你快速了解这两个概念。

[Rust入坑指南：朝生暮死](https://jackeyzhe.github.io/2020/03/02/Rust入坑指南：朝生暮死/)

[Rust入坑指南：智能指针](https://jackeyzhe.github.io/2020/03/09/Rust入坑指南：智能指针/)

接着是并发编程，Rust声称的安全并发，究竟是怎么保证的？

[Rust入坑指南：齐头并进（上）](https://jackeyzhe.github.io/2020/03/15/Rust入坑指南：齐头并进（上）/)

[Rust入坑指南：齐头并进（下）](https://jackeyzhe.github.io/2020/03/23/Rust入坑指南：齐头并进（下）/)

Safe Rust有这样那样的限制，有的开发者可能会觉得束手束脚，难以发挥实力。这时就可以考虑看看Unsafe Rust了。

[Rust入坑指南：居安思危](https://jackeyzhe.github.io/2020/03/31/Rust入坑指南：居安思危/)

最后是Rust的元编程，我们从最开始就在使用的`println!`宏，它是如何定义的呢？我们又怎么定义自己的宏？希望这篇文章对你有帮助。

[Rust入坑指南：万物初始](https://jackeyzhe.github.io/2020/04/08/Rust入坑指南：万物初始/)

经过这几个月的学习，我对Rust也有了一个初步的了解，在这里要感谢对我的分享提出意见的同学。也希望我的分享能对大家有所帮助。

虽然标题叫最后一舞，但是后面我还是会继续保持学习，也会不定期分享一些入门的代码案例给大家。

Rust编程，我们后会有期。