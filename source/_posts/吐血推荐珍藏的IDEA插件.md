---
title: 吐血推荐珍藏的IDEA插件
date: 2020-02-10 13:01:05
tags: 技术杂谈
---

之前给大家推荐了一些我自己常用的VS Code插件，很多同学表示很受用，并私信我说要再推荐一些IDEA插件。作为一名职业Java程序员/业余js开发者，我平时还是用IDEA比较多，所以也确实珍藏了一些IDEA插件。今天就一并分享给大家。<!-- more -->

在最开始，我还是想先介绍一下IDEA中如何安装插件，首先打开Preferences（菜单栏打开或者使用快捷键Command+,）在Windows版本中应该是Settings。然后选择Plugins一栏，就可以从右侧的MarketPlace中选择自己需要的插件进行安装了。

![idea插件](https://res.cloudinary.com/dxydgihag/image/upload/v1581059914/Blog/Other/idea_plugins/idea.png)

### Lombok

首先向我们走来的是Lombok。作为Java程序员，你还在为不断的写Getter/Setter方法而苦恼吗？你还在为每个Model类都要写类似的构造方法而感到烦恼吗？赶快试试Lombok吧，它可以有效帮助你解决这些问题，只需要一个注解，构造方法和Getter/Setter方法全部搞定，再也不用把时间浪费在无用功上了。

如果你还不是很了解Lombok的话，可以自己动手，到[Lombok官网](https://projectlombok.org/)学习一番，学完记得回来点赞。

最后展示一个简单的例子供大家参考。

![lombok](https://res.cloudinary.com/dxydgihag/image/upload/v1581059921/Blog/Other/idea_plugins/idea1.png)

### String Manipulation

String Manipulation插件是一款非常强大的插件，它可以对代码进行很多操作，如排序、去除空行、字符串格式转换、Encode/Decode。其中我最常用的是字符串格式转换。你可以通过点击右键选择String Manipulation或者使用快捷键Option + M来选择相应的功能。

![String Manipulation](https://res.cloudinary.com/dxydgihag/image/upload/v1581061347/Blog/Other/idea_plugins/idea2.gif)

### stackoverflow

作为一名高级CtrlCV工程师，我写代码有两大利器，一个是Google，另一个就是stackoverflow。两者相辅相成，帮我在编码的道路上越走越远。相信有不少同学跟我一样离不开stackoverflow，那么这款插件就会给你带来极大的方便，遇到问题怎么办？右键一下，点击「search stackoverflow」，大部分问题都可以轻松搞定。

### Rainbow Brackets

在推荐VS Code的插件时我们就介绍过一款叫做Bracket Pair Colorizer的插件，它可以把括号变成不同的颜色，我觉得这样分辨括号非常方便，看起来也比较美观。所以在IDEA中也使用了相同效果的插件，就是Rainbow Brackets。

![Rainbow Brackets](https://res.cloudinary.com/dxydgihag/image/upload/v1581062247/Blog/Other/idea_plugins/idea3.png)

### GsonFormat

我们在接外部接口时，别人给了一串JSON串，我们在代码中需要将JSON中的字段封装到一个类中，一个一个输入挺麻烦的，这时GsonFormat就可以派上用场了。它可以帮助我们根据JSON中的key快速生成我们需要的类。

它的使用快捷键是Option + S

![GsonFormat](https://res.cloudinary.com/dxydgihag/image/upload/v1581066692/Blog/Other/idea_plugins/idea4.gif)

### Maven Helper

如果你的项目使用的构建工具是Maven的话，这个插件就能帮你避免各种依赖冲突，安装好插件之后，打开pom文件，可以看到最下方有一个叫Dependency Analyzer的Tab，这里就可以看到你的哪些依赖是有冲突的，然后在右侧Exclude掉不需要的依赖。

![Maven Helper](https://res.cloudinary.com/dxydgihag/image/upload/v1581063616/Blog/Other/idea_plugins/idea5.png)

### RestfulToolkit

RestfulToolkit是一套辅助开发Restful服务的工具集，对于这个插件，我最常用的功能就是快速查找指定的url对应的方法。快捷键是Command + \

关于其他的一些功能，大家有兴趣的话可以直接访问该插件的[homepage](https://plugins.jetbrains.com/plugin/10292-restfultoolkit)。

以上这些就是我常用的IDEA插件了，没有太多花里胡哨的东西，大家如果有什么好用的插件也欢迎分享出来。