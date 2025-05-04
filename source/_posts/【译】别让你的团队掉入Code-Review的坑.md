---
title: 【译】别让你的团队掉入Code Review的坑
date: 2019-09-12 23:31:26
tags: Code Review
---

代码审查是许多高效团队的工程实践。即使你的软件已经有很多优点了，但团队在做代码审查时仍然会遇到一些陷阱。<!-- more -->

这篇文章我讲向你介绍一些需要特别注意的陷阱，以防代码审查工作拖累你的团队。知道可能遇到的问题或陷阱，将会帮助你进行更加高效、有效的的代码审查工作。这是我们[调查了900名微软员工后得到的结论](https://www.michaelagreiler.com/wp-content/uploads/2019/03/Code-Reviewing-in-the-Trenches-Understanding-Challenges-Best-Practices-and-Tool-Needs.pdf)。

#### 一个典型的代码审查过程

一个典型的使用工具进行的代码审查过程大致是这样的：开发者完成一段代码，她提交代码准备开始让别人review。然后她选择需要审查她的代码的人。审查代码的人开始查看代码并给出一些评论。作者按照这些评论完善代码。当所有人都觉得没问题了以后，代码就可以合并进代码库了。在另一篇中我详细介绍了微软的代码审查的过程。

我为我的邮件订阅读者们准备了一份代码审查电子书，书中包含了全部的代码审查最佳实践的清单。我还加了一些额外的见解，大家可以[自行申请](https://www.michaelagreiler.com/code-review-e-book/)。

![Check out the VIP [Code Review e-Book](https://www.michaelagreiler.com/code-review-e-book/) I prepared for my subscribers.](https://i1.wp.com/www.michaelagreiler.com/wp-content/uploads/2019/05/E-Book-Preview.jpg?w=860&ssl=1)

#### 代码审查并不总是一个平稳的过程

这些步骤听起来像是 一个很平稳的过程，其实并不是，像其他所有事情一样，实际过程往往不如预期。在代码审查过程中，经常会遇到一些陷阱，这会降低整个审查代码的积极性。如果不能正确处理，代码审查会对整个团队的工作效率产生影响。所以让我们来看一下代码审查过程中究竟存在哪些坑。

关键问题主要有两种：代码审查花费的时间和代码审查所能提供的价值。

#### 等待回复是一种煎熬

作者需要面对的一个最主要的陷阱就是及时收到回复。等待评论的过程中不能在代码中做其他工作是一个巨大的问题。即使开发者可以完成其他任务，如果代码审查工作耗时过长，也会对开发者对工作效率和满意度造成不好的影响。

#### 开发者必须兼顾多项职责

代码审查并不是代码审查人员唯一的任务。相反，它只是开发者日常工作之一（即使需要每天花费大量时间来做）。所以代码审查人员很可能在做其他工作，并且必须在开始代码审查之前停下或完成这些工作。

如果时间不理想，或者代码审查人员在之前没有预料到这种变化，那么她可能在代码审查之前需要一点时间。远程团队还需要注意时差，这意味着代码审查可能花费更多的时间。

#### 开发者会面对代码审查不被当作实际工作的问题

对于作者和代码审查人员来说，时间都是有限的，如果团队想要开发者做代码审查工作，但又不把它计入工作时间，这是有很大问题的。

![You have to value and plan for the time spent doing code reviews.
Photo by [freestocks.org](https://unsplash.com/photos/vcPtHBqHnKk?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)](https://i1.wp.com/www.michaelagreiler.com/wp-content/uploads/2019/04/Code-Review-value-time.png?w=1024&ssl=1)

#### 代码审查工作和效果没有奖励

如果只是声称对代码审查工作进行评估而没有任何奖励，这将没有意义。大多数公司只会对开发者开发的功能或者写出的代码进行奖励，这减弱了开发人员在工作中帮助他人的动力和能力。代码审查应该成为绩效评估和晋升决策的基石。

#### 社会因素和团队动力

等待代码审查并不总是和缺乏时间以及缺少奖励系统有关。由于社会性质，延迟审查可能是由于不安全感或团队动力。尤其是如果代码审查工作是压倒性的或者审查者是新手，那么代码审查会让人不知所措。

#### 大量代码很难审查

大量的代码是审查工作的另一个巨大的陷阱。想象一下如果你是代码审查人员，当你需要审查一段代码时，你通常会想：我很快就能看完，但是当你打开的时候，发现这是一段巨大的改动，修改遍布整个代码库，你的第一反应是什么？

大概是：天呐！（译者注：也可能是我*！）

没错，这正是我们在分析了数千次代码评审中看到的。随着代码数量的增加，不仅是审核时间需要增加，评论的质量也会随之下降。这也是可以理解的。

![tweet](https://res.cloudinary.com/dxydgihag/image/upload/v1567928096/Blog/Other/code_review/code-review00.png)

大量的代码更改非常难以审核，另外，如果审核人员对这部分代码并不熟悉，那这将是一场噩梦。

#### 理解代码修改需要一些指导

大多数审核人员会面临无法理解代码修改的动机的困境。如果没有任何对修改的描述和解释，代码审查工作将会非常难以进行。研究表明如果审核人员不理解代码修改的目的，或者由于量大而不堪重负，那么她很难给出良好的建议。

> It’s just this big incomprehensible mess… then you can’t add any value because they are just going to explain it to you and you’re going to parrot back what they say.

#### 没有得到有价值的反馈会降低开发人员审核代码的动力

毫无疑问，花费时间去等待代码审查，却没有得到有用的反馈是一个常见的问题。虽然团队可能从知识分享中受益，但这样仍会降低开发者的动力和收益。

审核人员不能给出有效建议的原因通常有这样几种

1. 审核人员不具备这方面专业知识
2. 审核人员没有时间去完整的审查代码
3. 审核人员没有理解修改的目的

#### 一旦主要讨论代码格式，你就需要采取行动

另一个在代码审查过程中很重要的问题叫做bikeshedding。它的意思是开发者专注于小问题的讨论（例如代码格式）而忽略了更严重的问题。导致这种行为的原因有很多，通常是由于审核人员并不理解改动的目的或者没有时间做代码审查。有时bikeshedding可能代表团队的动力出了问题。

#### 达成共识可能需要面对面讨论

有时候会发生代码作者和审核人员或者是审核人员之间很难达成共识的情况。由于团队动力和这件事紧密相关，因此必须小心处理这种情况通过工具和书面形式只会加剧这种问题。如果要讨论有争议的问题，那么面对面或者视频也许是更好的方案。

#### 代码审核的好处大于工作量

我希望这些陷阱不会改变你对代码审核的想法，因为如果你了解了这些陷阱并妥善处理它们，代码审核工作对于整个技术团队是非常有好处的，而且会有更有效的方法来做代码审核。

#### 原文地址

https://www.michaelagreiler.com/code-review-pitfalls-slow-down/

最后，如果可以的话请帮忙填一下调查问卷。

![调查问卷](https://res.cloudinary.com/dxydgihag/image/upload/v1568307144/Blog/Other/code_review/code-review-qrcode.jpg)