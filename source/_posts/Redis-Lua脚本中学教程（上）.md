---
title: Redis Lua脚本中学教程（上）
date: 2019-06-10 19:12:35
tags: Redis
comments: true
---

失踪人口回来啦！

有读者问我为什么这么久都没有出Redis Lua中学教程，表示村头厕所已经好久没有纸了。其实我早就要写这篇中学教程了，奈何最近太忙了，就一拖再拖，直到今天我终于又开始动笔了。忘记Lua相关概念的同学可以先回顾一下[小学教程](https://jackeyzhe.github.io/2019/05/13/Redis-Lua%E8%84%9A%E6%9C%AC%E5%B0%8F%E5%AD%A6%E6%95%99%E7%A8%8B/)。<!-- more -->

中学教程主要分为两部分：Redis Lua的相关命令详解和Lua的语法介绍。

前面我们简单介绍了EVAL和EVALSHA命令。但是只有那点只是是没办法从中学毕业的，因此我们需要进行更深入的学习。

#### EVAL

最早可用版本：2.6.0

用法：EVAL script numkeys key [key ...] arg [arg ...]

关于用法我们已经演示过了，其中第一个参数是要执行的Lua脚本，第二个参数是传入脚本的参数个数。后面则是参数的key数组和value数组。

在Lua中执行Redis命令的方法我们也介绍过，就是使用redis.call()和redis.pcall()两个函数。它们之间唯一的不同就是当Redis命令执行错误时，redis.call()会抛出这个错误，使EVAL命令抛出错误，而redis.pcall()会捕获这个错误，并返回Lua的错误表。

通常我们约定执行命令的key都需要由参数传入，命令必须在执行之前进行分析，以确定它作用于哪个key。这样做的目的是为了在一定程度上保证EVAL执行的Lua脚本的正确性。

##### Lua和Redis之间数据类型的转换

在Redis执行EVAL命令时，如果脚本中有call()或者pcall()命令，就会涉及到Redis和Lua之间数据类型转换的问题。转换规则要求，一个Redis的返回值转换成Lua数据类型后，再转换成Redis数据类型，其结果必须和初始值相同。所以每种类型是一一对应的。转换规则如下：

##### Redis与Lua互相转换

|          Redis           |               Lua               |
| :----------------------: | :-----------------------------: |
|         integer          |             number              |
|           bulk           |             string              |
|        multi bulk        |              table              |
|          status          | table with a single `ok` field  |
|          error           | table with a single `err` field |
| Nil bulk &Nil multi bulk |       false boolean type        |

除此之外，Lua到Redis的转换还有一些其他的规则：

- Lua boolean true -> Redis integer reply with value of 1
- Lua只有一种数字类型，不会区分整数和浮点数。而数字类型只能转换成Redis的integer类型，如果要返回浮点数，那么在Lua中就需要返回一个字符串。
- Lua数组在转换成Redis类型时，遇到nil就停止转换

来个栗子验证一下：

``` bash
EVAL "return {1,2,3.3333,'foo',nil,'bar'}" 0
1) (integer) 1
2) (integer) 2
3) (integer) 3
4) "foo"
```

可以看到bar没有返回，并且3.333返回了3。

##### 脚本的原子性

Redis运行所有的Lua命令都使用相同的Lua解释器。当一个脚本正在执行时，其他的脚本或Redis命令都不能执行。这很像Redis的事务multi/exec。这意味着我们要尽量避免脚本的执行时间过长。

##### 脚本整体复制

当脚本进行传播或者写入AOF文件时，Redis通常会将脚本本身进行传播或写入AOF，而不是使用它产生的若干命令。原因很简单，传播整个脚本要比传播一大堆生成的命令的速度要快。

从Redis3.2开始，可以只复制影响脚本执行结果的语句，而不用复制整个脚本。这个复制整个脚本的方法有以下属性：

- 如果输入相同，脚本必须输出相同的结果。即执行结果不能依赖于隐式的变量，或依赖于I/O输入
- Lua不会导出访问系统时间或其他外部状态的命令
- 如果先执行了“随机命令”（如RANDOMKEY，SRANDMEMBER，TIME），并改变了数据集，接着执行脚本时会被阻塞。
- 在Redis4中，Lua脚本调用返回随机顺序的元素的命令时，会在返回之前进行排序，也就是说，调用redis.call("smembers",KEYS[1])，每次返回的顺序都相同。从Redis5开始就不需要排序了，因为Redis5复制的是产生影响的命令。
- Lua修改了伪随机函数math.random和math.randomseed，使每次执行脚本时seed都相同，而如果不执行math.randomseed，只执行math.random时，每次的结果也都相同。

##### 复制命令队列

在这种模式下，Redis在执行脚本时会收集所有影响数据集的命令，当脚本执行完毕时，命令队列会被放在事务中，发送给AOF文件。

Lua可以通过执行redis.replicate_commands()函数来检查复制模式，如果返回true表示当前是复制命令模式，如果返回false，则是复制整个脚本模式。

##### 可选择的复制命令

脚本复制模式选择好以后，就可以对复制到副本和AOF的方式进行更多的控制。这是一种高级特性，因为滥用会切断主从备份，和AOF持久化。如果我们只需要在master上执行某些命令时，这一特性就变得很有用。例如我们需要计算一些中间值时，只需要在master上计算就好，那么这些命令就不必进行复制。

从Redis3.2开始，有一个新的命令叫做redis.set_repl()，它可以用来控制复制方式，有如下选项（默认是REPL_ALL）：

``` bash
redis.set_repl(redis.REPL_ALL) -- Replicate to AOF and replicas.
redis.set_repl(redis.REPL_AOF) -- Replicate only to AOF.
redis.set_repl(redis.REPL_REPLICA) -- Replicate only to replicas (Redis >= 5)
redis.set_repl(redis.REPL_SLAVE) -- Used for backward compatibility, the same as REPL_REPLICA.
redis.set_repl(redis.REPL_NONE) -- Don't replicate at all.
```

##### 全局变量

为了避免数据泄露，Redis脚本不允许创建全局变量。如果必须有一个公共变量，可以使用Redis的key来代替。在EVAL命令中创建一个全局变量会引起一个异常。

``` bash
> eval 'a=10' 0
(error) ERR Error running script (call to f_933044db579a2f8fd45d8065f04a8d0249383e57): user_script:1: Script attempted to create global variable 'a
```

##### 关于SELECT的使用

在Lua脚本中使用SELECT就像在正常客户端中使用一样。值得一提的是，在Redis2.8.12之前，Lua脚本中执行SELECT是会影响到客户端的，而从2.8.12开始，Lua脚本中的SELECT只会在脚本执行过程中生效。这点在Redis版本升级时需要注意，因为升级前后，命令的语义会改变。

##### 可用的库

Lua脚本中有许多库，但并不是都能在Redis中使用，其中可以使用的有：

- `base` lib.
- `table` lib.
- `string` lib.
- `math` lib.
- `struct` lib.
- `cjson` lib.
- `cmsgpack` lib.
- `bitop` lib.
- `redis.sha1hex` function.
- `redis.breakpoint and redis.debug` function in the context of the [Redis Lua debugger](https://redis.io/topics/ldb).

struct, CJSON and cmsgpack是外部库，其他的都是Lua的标准库。

##### 在脚本中打印Redis日志

使用redis.log(loglevel,message)函数可以在Lua脚本中打印Redis日志。

loglevel包括：

- redis.LOG_DEBUG
- redis.LOG_VERBOSE
- redis.LOG_NOTICE
- redis.LOG_WARNING

它们与Redis的日志等级是对应的。

##### 沙箱和最大执行时间

脚本不应该访问外部系统，包括文件系统和其他系统。脚本应该只能操作Redis数据和传入进来的参数。

脚本默认的最大执行时间是5秒（正常脚本执行时间都是毫秒级，所以5秒已经足够长了）。可以通过修改lua-time-limit变量来控制最大执行时间。

当脚本执行时间超过最大执行时间时，并不会被自动终止，因为这违反了脚本的原子性原则。当一个脚本执行时间过长时，Redis会有如下操作：

- Redis记录下这个脚本执行时间过长
- 其他客户端开始接收命令，但是所有的命令都会会返回繁忙，除了SCRIPT KILL 和 SHUTDOWN NOSAVE
- 如果一个脚本仅执行只读命令，则可以用SCRIPT KILL命令来停止它。
- 如果脚本执行了写入命令，那么只能用SHUTDOWN NOSAVE来终止服务器，当前的所有数据都不会保存到磁盘。

#### EVALSHA

最早可用版本：2.6.0

用法：EVALSHA sha1 numkeys key [key ...] arg [arg ...]

该命令用来执行缓存在服务器上的脚本，sha1为脚本的唯一标识。

使用EVAL命令必须每次都要把脚本从客户端传到服务器，由于Redis的内部缓存机制，它并不会每次都重新编译脚本，但是传输上仍然浪费带宽。

另一方面，如果使用特殊命令或者通过redis.conf来定义命令会有以下问题：

- 不同实例有不同的实现方式
- 发布将会很困难，特别是分布式环境，因为要保证所有实例都包含给定的命令
- 读应用程序代码时，由于它调用了服务端命令，会不清楚代码的语义

为了避免这些问题，同时避免浪费带宽，Redis实现了EVALSHA命令。

如果服务器中没有缓存指定的脚本，会返回给客户端脚本不存在的错误信息。

#### SCRIPT DEBUG

最早可用版本：3.2.0

时间复杂度：O(1)

用法：SCRIPT DEBUG YES|SYNC|NO

该命令用于设置随后执行的EVAL命令的调试模式。Redis包含一个完整的Lua调试器，代号为LDB，可以使编写复杂脚本的任务更加简单，在调试模式下，Redis充当远程调试服务器，客户端可以逐步执行脚本，设置断点，检查变量等。想了解更多调试器内容的可以查看官方文档[Redis Lua debugger](https://redis.io/topics/ldb)。

LDB可以设置成异步或同步模式。异步模式下，服务器会fork出一个调试会话，不会阻塞主会话，，调试会话结束后，所有数据都会回滚。同步模式则会阻塞会话，并保留调试过程中数据的改变。

#### SCRIPT EXISTS

最早可用版本：2.6.0

时间复杂度：O(N)，N是脚本数量

返回脚本是否存在于缓存中（存在返回1，不存在返回0）。这个命令适合在管道前执行，以保证管道中的所有脚本都已经加载到服务器端了，如果没有，需要用SCRIPT LOAD命令进行加载。

#### SCRIPT FLUSH

最早可用版本：2.6.0

时间复杂度：O(N)，N是缓存中的脚本数

刷新缓存中的脚本，这一命令常在云服务上被使用。

#### SCRIPT KILL

最早可用版本：2.6.0

时间复杂度：O(1)

停止当前正在执行的Lua脚本，通常用来停止执行时间过长的脚本。停止后，被阻塞的客户端会抛出一个错误。

#### SCRIPT LOAD

最早可用版本：2.6.0

时间复杂度：O(N)，N是脚本的字节数

该命令用于将脚本加载到服务器端的缓存中，但不会执行。加载后，服务器会一直缓存，因为良好的应用程序不太可能有太多不同的脚本导致内存不足。每个脚本都像一个新命令的缓存，所以即使是大型应用程序，也就有几百个，它们占用的内存是微不足道的。

#### 小结

本文介绍了Redis Lua相关的命令。其中EVAL和EVALSHA用来执行脚本。脚本执行具有原子性。脚本的复制和传播可以根据需要设置。脚本中不能定义全局变量。