---
title: 代码检查又一利器：ArchUnit
date: 2019-12-16 23:28:09
tags: 整洁
---

Code Review总是让人又爱又恨，它可以帮助我们在提测之前发现很多代码中比较“丢人”的问题，但是，Code Review通常会比写代码更加耗费精力，因为你需要理解别人的代码，而为了这一目的，往往需要很多次的沟通。<!-- more -->

人们常说“见字如面”。我认为代码也是一样，看到一个人的代码，就会对这个人有一个大概的印象。例如，当你看到一段代码写的非常随意，随意的格式、随意的命名、随意的封装，然后又没有单元测试，那我们一般会认为这段代码的作者是一个不够严谨、做事随意、有些懒惰，又对自己的代码责任心不强的人。如果你不是这样的人，那就需要花费更多的力气向同事证明自己。而如果在代码中做好每一个细节，严格遵循编码规范，单元测试覆盖率比较高，那么同事对你的第一印象一定是这个人还是比较可靠的，跟他合作应该比较愉快。

说了这么多，其实就是想强调Code Review的重要性。那么既然它这么重要，但又给我们带来了更大的工作量。作为程序员，我们一定会想，能不能自动化？答案当然是可以。事实上现在也有很多公司实现了自动化，例如自动进行静态代码分析来确保代码质量，利用类似[Cobertura](https://cobertura.github.io/cobertura/)这样的工具来检查单元测试覆盖程度等等。但是这并不能完全保证代码的整洁性和可靠性。

有了这些工具之后Code Review轻松了许多，但是这些工具的安装、使用也是需要花费很高的成本的。所以我想给大家介绍的是一个使用简单、方便的工具来帮我完成这些任务。在介绍之前，我们先来想一想我们平时在Review别人代码时可能会注意哪些问题。这里我简单列出来了一些：

- 抛出的异常不能太过广泛
- 不能写``` System.out ```，而是要用日志输出
- 不能使用```java.util.logging```
- 如果使用贫血模型开发，每个类需要放到对应的包中
- 接口不能放在实现类的包中
- ```Service```层代码不能访问```Controller```层代码
- 合理使用第三方库

这些事情以前我们都是靠人工来检查，直到我发现了[ArchUnit]( https://www.archunit.org/ )这个库。感觉像是抓住了自动化道路上的救命稻草。

### 什么是ArchUnit？

ArchUnit的官方网站是 [https://www.archunit.org](https://www.archunit.org/) 

官网中原话介绍是

>  ArchUnit is a free, simple and extensible library for checking the architecture of your Java code using any plain Java unit test framework.

意思是ArchUnit是一款免费、简单可扩展的库，它可以使用任何Java单元测试框架来检查Java代码的架构。

也就是说，它的主要功能是用来检查代码结构的。那么怎么使用呢？

### 如何使用？

ArchUnit的简单绝对不是空谈，如果你是maven项目，只需要在pom.xml文件中添加如下依赖：

``` xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit</artifactId>
    <version>0.12.0</version>
    <scope>test</scope>
</dependency>
```

如果你是Gradle项目，使用起来同样非常简单

``` 
dependencies {
    testCompile 'com.tngtech.archunit:archunit:0.8.0'
}
```

当你添加了依赖以后，就可以为我们前面提到的规则写测试用例了。

当然，也有一些内建的通用规则，它们定义在

``` java
com.tngtech.archunit.library.GeneralCodingRules
```

这个类中。关于内建规则的细节，可以查看[官方文档]( https://www.archunit.org/userguide/html/000_Index.html#_general_coding_rules )。

### 自定义规则

除了内建规则以外，ArchUnit也支持你定义自己需要的规则，至于如何定义规则，文档中都有详细的介绍。当然，也可以参考这个例子来写一些规则。 https://github.com/TNG/ArchUnit-Examples

### 如何执行

规则定义好以后如何执行呢？我们说ArchUnit使用起来非常简单，如果需要测试，对maven项目来说只需要执行命令

``` bash
mvn test
```

而对于Gradle项目来说，只要执行命令

``` bash
gradle test
```

### 总结

ArchUnit看起来是一个很酷的三方库，我并没有在使用层面做过多介绍，因为我也在摸索中，感兴趣的朋友可以和我一起交流。