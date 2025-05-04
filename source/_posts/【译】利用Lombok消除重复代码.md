---
title: 【译】利用Lombok消除重复代码
date: 2019-11-20 21:47:47
tags: Java
---

当你在写Getter和Setter时，一定无数次的想过，为什么会有POJO这么烂的东西。你不是一个人！（不是骂人…）无数的开发人员花费了大量的时间来写这种样板代码，而他们本来可以利用这些时间做出更有价值的输出。<!-- more -->

从我开始写Java以来，已经写了几千行代码了，其中大概50%都是样板代码，在转型之前，我就这么一直毫无怨言的写着。而最近两年，我不再Java了，转而开始写一些Python，Go和JavaScript的代码。这时我才感觉到Java中的重复的样板代码是多么令人沮丧。

值得庆幸的是，现在的IDE为我们提供了自动生成这些代码的功能。但是我仍然需要按快捷键或者点鼠标来操作，这是非常影响我的编码思路的。

### Lombok简介

> [Project Lombok](https://projectlombok.org/) *is a java library that automatically plugs into your editor and build tools, spicing up your java. Never write another getter or equals method again*

上面这段话摘自Lombok的[首页](https://projectlombok.org/)。这是一个每个人都需要使用的库，简直是一种仙丹！开个玩笑。Java是一门很棒的语言，但是它的冗长经常会令人感到苦恼。

Lombok到底有多香呢？我总结了以下几点：

1. Getter和Setter注解会自动生成getter、setter方法
2. NoArgsConstructor和AllArgConstructor可以帮助你快速生成构造函数
3. ToString会使POJO打印更加友好的日志
4. Data会让你的POJO成为一个完全符合规范的POJO
5. SneakyThrows可以偷偷抛出检查异常，而不需要再写throws子句

想了解更多Lombok特性的话，可以自行前往https://projectlombok.org/features/all查看。

### Lombok是如何工作的？

Lombok是在Java注解处理器和几个编译时注解的帮助下工作的，它将注入额外的Java字节码来帮助我们处理重复的代码。你可以查看它生成的Java代码，这一过程被幽默的称为“Delombokisation”。

### 我应该如何开始使用？

Lombok引入了一个额外的编译时依赖。

如果你使用vanilla javac进行编译，你需要指定[lombok.jar](https://projectlombok.org/download)作为注解处理器：```javac -cp lombok.jar MyCode.java```

如果你使用的是maven，那么需要在pom.xml中插入以下代码来保证你的代码可以使用Lombok。

``` xml
<dependencies>
	<dependency>
		<groupId>org.projectlombok</groupId>
		<artifactId>lombok</artifactId>
		<version>1.18.4</version>
		<scope>provided</scope>
	</dependency>
</dependencies>
```

如果你使用的是Gradle，那么你需要使用[Gradle Lombok插件](https://github.com/franzbecker/gradle-lombok)

``` 
plugins {
    id 'io.franzbecker.gradle-lombok' version '1.14'
    id 'java'
}
```

#### 设置你的IDE

从你开始使用Java起，你应该就开始使用一个智能的IDE来自动编译或给你的代码提供一些建议。为了将Lombok集成进IDE，你需要告诉Lombok io来安装合适的钩子。

获取Lombok的jar包后，执行``` java -jar lombok.jar```来完成所有的设置。

IntelliJ IDEA和Visual Studio用户需要一个单独的Lombok插件，你可以选择从插件库中安装。

### 代码拿来！

talk is cheap, show me your code.程序员就应该拿代码说话。下面我们就来看一个完整的例子。

#### Getters和Setters

为被注解的自动生成getXXX和setXXX方法。

``` java
import lombok.Getter;
import lombok.Setter;

class UptimeResponse {
    // GetXXX and SetXXX are automatically generated
    @Getter @Setter private long uptime;
    @Getter @Setter private long currentTime;
    @Getter @Setter private String status;
    UptimeResponse() {
        this.uptime = ManagementFactory
                          .getRuntimeMXBean().getUptime();
        this.currentTime = System.currentTimeMillis();
        this.status = "OK";
    }
}
// So this works automagically
UptimeResponse res = new UptimeResponse();
res.setStatus("FAIL");
System.out.println(res.getUptime());
```

#### Constructors

可以自动创建默认的POJO构造函数，它将字段初始化为默认值。

1. NoArgConstructor创建一个无参构造函数，所有的字段都会初始化为默认值
2. AllArgsConstructor创建一个全参数构造函数，每个字段都会初始化为指定值
3. RequiredArgsConstructor创建一个构造函数，参数包括所有final字段和标记为NotNull的字段

``` java
import lombok.*

@AllArgsConstructor
class Document {
    @Getter @Setter private String title;
    @Getter @Setter private String content;
    // ...
}
// This works automagically
Document d = new Document("Hello World", "Message Body");
d.getTitle();   // Hello World
d.getContent(); // Message Body
```

#### Equals and hash codes

Lombok可以生成的样板代码是包含局部变量的equals方法和hashcode方法。你可以手动排除一些字段。

``` java
import lombok.*;

@RequiredArgsConstructor
@EqualsAndHashCode
class User {
    @Getter
    private final String username;
    @EqualsAndHashCode.Exclude
    @Getter
    @Setter
    private String lastAction;  // not required for equality checks
}
// This works automagically
User u1 = new User("amitosh");
u1.setLastAction("Hello");
User u2 = new User("amitosh");
u2.setLastAction("Compile");
u1.equals(u2) // Gives true
```

#### To String

Lombok的ToString注解自动生成toString方法，其中包含类封装的全部字段。这是用于生成调试表示的快速方法。

``` java
import lombok.ToString;
import lombok.Getter;
import lombok.Setter;

@ToString
class Entry {
    @Getter @Setter private String id;
    @Getter @Setter private String target;
}
// This works automagically
Entry e = new Entry();
// ...
System.out.println(e);  // Nice output with values of id and target
```

#### Data classes

这个注解用于生成符合规范的完整POJO。它是ToString、EqualsAndHashCode以及所有非final字段的Getter和Setter的集合体。

``` java
import lombok.Data;

@Data
class Message {
    private String sender;
    private String content;
}
// This works automagically
Message m = new Message("amitosh", "Hello World");
m.setSender("agathver");
m.getContent();  // Hello World
m.toString();    // ...
```

#### SneakyThrows

Java是一门静态检查语言，但有时检查会比较多余。例如有时我们不关心异常，或者确定代码中不会出现异常，所以就不想去写捕获和处理异常的代码。这时SneakyThrows注解可以帮助我们一起骗过编译器。

但要注意不能滥用这个注解。

``` java
import lombok.SneakyThrows;
 
public class SneakyThrowsExample {
   @SneakyThrows(UnsupportedEncodingException.class)
   public String utf8ToString(byte[] bytes) {
       // This exception is never generated as UTF-8 is guaranteed
       // to be supported by the JVM
       return new String(bytes, "UTF-8");
   }
}
```

### Delomboking

不是所有的工具都支持Lombok的，最著名的是JavaDoc工具。你需要有一个中间态的代码来使文档正确表示。此外，有时候你可能会想看看Lombok生成的代码到底是什么样的。幸好Lombok提供了“delomboking”，用来将Lombok转换成Java源代码。

要转换一个文件夹下的全部代码，可以使用以下命令：

``` bash
java -jar lombok.jar delombok src -d src-delomboked
```

maven和gradle插件也包含了delomboking任务，在你需要的时候可以使用。

Lombok是一个提高你的Java生产力的工具。我无法想象没有它时应该怎么写Java程序。真心希望你在读完本文以后能够认识到它的强大！

### 原文地址

https://medium.com/@agathver/banish-repetitive-java-code-with-lombok-f9b97d0d4137

### 译者点评

Lombok是一款非常好用的工具，它可以帮助我们快速构建POJO类。但是如果直接使用@Data注解时，会破坏类的封装特性。这点不符合面向对象编程的思想，但工作中会使用一些序列化工具，这些工具要求所有字段都要有setter方法。为了编码的方便，可能使用@Data方法是一个好的选择。

