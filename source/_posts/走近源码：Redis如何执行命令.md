---
title: 走近源码：Redis如何执行命令
date: 2019-01-05 17:23:28
tags: Redis
---

[前文](https://jackeyzhe.github.io/2019/01/04/%E8%B5%B0%E8%BF%91%E6%BA%90%E7%A0%81%EF%BC%9ARedis%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B/)我们了解了Redis的启动过程。在initServer()函数中创建了一些循环事件来监听TCP端口和Unix的Sockets，从而使Redis服务器可以接收新的连接。今天我们再一起来看一下Redis究竟是如何处理命令并返回结果的。<!-- more -->

##### 处理新连接

Redis在initServer()函数中创建循环事件调用了acceptTcpHandler和acceptUnixHandler函数（都在networking.c文件中）来处理接收到的TCP连接和Unix的Sockets连接。这两个函数又调用了acceptCommonHandler()函数，在这个函数中调用了createClient()函数创建一个新的client对象，用来表示一个新的客户端连接。

createClient()函数具体做了哪些事情呢？

首先为变量c分配了内存，接着将Socket连接置为非阻塞状态，并且设置了TCP无延迟。然后创建了File循环事件（aeCreateFileEvent）来调用readQueryFromClient函数。新建的客户端默认连接的是服务器的第一个数据库（编码为0），最后需要设置好客户端的各种属性和状态。

##### 读一个客户端的命令

刚刚我们提到了readQueryFromClient函数，从名称上就能看出来这个函数是用来从客户端读取命令的。下面来看看函数的具体实现。

Redis会先将命令读入缓冲区，一次最多读取的大小是PROTO_IOBUF_LEN（1024*16）bit。然后调用processInputBufferAndReplicate()函数，来处理缓冲区中的数据，如果客户端是master（主从同步过程），那么Redis会计算处理前后缓冲区的不同部分，以确定从节点接收了多少数据。processInputBufferAndReplicate()函数会处理客户端向服务器发送命令和主节点向从节点发送命令这两种情况，不过最后都需要调用processInputBuffer()函数。

processInputBuffer()函数会先判断客户端是否正常，如果出现连接中断或者客户端阻塞等情况，就会立即停止处理命令，不做无用功。然后根据读取的请求生成相应的Redis可以执行的命令（包括参数）。不同的请求类型分别调用processInlineBuffer()和processMultibulkBuffer()函数。生成好命令之后，交给processCommand()（server.c文件中）函数执行，如果返回C_OK则重置客户端，等待下一个命令。如果返回的是C_ERR，则客户端会被销毁（比如执行QUIT命令）。

processCommand()函数会从Redis启动时加载的命令表中查找命令，然后检查命令的执行权限。

如果是cluster，这时会判断key是否属于当前的master，不属于需要返回重定向信息。

如果内存不够用，这里也需要判断一下是否有可以释放的内存，如果没有，就不能执行命令，返回错误信息。

接下来会判断一些不能接收写命令的情况：

- 服务器不能进行持久化
- 作为master，没有足够的可用的slave
- 此服务器为只读的slave，只有它的master可以接收写命令

在订阅模式中，只能接收指定的命令：(P)SUBSCRIBE / (P)UNSUBSCRIBE / PING / QUIT。

当slave和master失联时，只能接收有flag "t"的命令，例如，INFO，SLAVEOF等。

如果命令没有CMD_LOADING标志，并且当前服务器正在加载数据，则不能接收此命令。

对lua脚本的长度进行限制。

进行完上面的各种条件判断之后，才可以真正开始调用call()函数执行命令。

##### 执行命令并返回

call()函数的参数是client类型的，取出cmd成员进行执行。

```c
/* Call the command. */
dirty = server.dirty;
start = ustime();
c->cmd->proc(c);
duration = ustime()-start;
dirty = server.dirty-dirty;
if (dirty < 0) dirty = 0;
```

如果是写命令，就会使服务器变“脏”，也就是服务器需要标记一下内存中的某些页有了改变。这对于Redis的持久化来说非常重要，它可以知道这个命令影响了多少个key。命令执行完之后并没有结束，call函数还会做一些其他操作。例如记录日志，写AOF文件，向从节点同步命令等。

至于返回值，每个命令有各自的处理方法，我们后面在介绍。

到这里，Redis处理命令的过程也就完成了。

后面我们会再通过具体的命令来对这个过程做一个更清晰的介绍。