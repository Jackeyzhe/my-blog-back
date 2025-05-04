---
title: 走近源码：Redis命令执行过程（客户端）
date: 2019-01-12 15:32:52
tags: Redis
---

前面我们了解过了当Redis执行一个命令时，服务端做了哪些事情，不了解的同学可以看一下这篇文章[走近源码：Redis如何执行命令](https://jackeyzhe.github.io/2019/01/05/%E8%B5%B0%E8%BF%91%E6%BA%90%E7%A0%81%EF%BC%9ARedis%E5%A6%82%E4%BD%95%E6%89%A7%E8%A1%8C%E5%91%BD%E4%BB%A4/)。今天就一起来看看Redis的命令执行过程中客户端都做了什么事情。<!-- more -->

#### 启动客户端

首先看redis-cli.c文件的main函数，也就是我们输入redis-cli命令时所要执行的函数。main函数主要是给config变量的各个属性设置默认值。比如：

- hostip：要连接的服务端的IP，默认为127.0.0.1
- hostport：要连接的服务端的端口，默认为6379
- interactive：是否是交互模式，默认为0（非交互模式）
- 一些模式的设置，例如：cluster_mode、slave_mode、getrdb_mode、scan_mode等
- cluster相关的参数

……

接着调用parseOptions()函数来处理参数，例如-p、-c、--verbose等一些用来指定config属性的（可以输入redis-cli --help查看）或是指定启动模式的。

处理完这些参数后，需要把它们从参数列表中去除，剩下用于在非交互模式中执行的命令。

parseEnv()用来判断是否需要验证权限，紧接着就是根据刚才的参数判断需要进入哪种模式，是cluster还是slave又或者是RDB……如果没有进入这些模式，并且没有需要执行的命令，那么就进入交互模式，否则会进入非交互模式。

``` c
/* Start interactive mode when no command is provided */
if (argc == 0 && !config.eval) {
    /* Ignore SIGPIPE in interactive mode to force a reconnect */
    signal(SIGPIPE, SIG_IGN);

    /* Note that in repl mode we don't abort on connection error.
     * A new attempt will be performed for every command send. */
    cliConnect(0);
    repl();
}

/* Otherwise, we have some arguments to execute */
if (cliConnect(0) != REDIS_OK) exit(1);
if (config.eval) {
    return evalMode(argc,argv);
} else {
    return noninteractive(argc,convertToSds(argc,argv));
}
```

#### 连接服务器

cliConnect()函数用于连接服务器，它的参数是一个标志位，如果是CC_FORCE（0）表示强制重连，如果是CC_QUIET（2）表示不打印错误日志。

如果建立了socket，那么就连接这个socket，否则就去连接指定的IP和端口。

``` c
if (config.hostsocket == NULL) {
    context = redisConnect(config.hostip,config.hostport);
} else {
    context = redisConnectUnix(config.hostsocket);
}
```

##### redisConnect

redisConnect()（在deps/hiredis/hiredis.c文件中）函数用于连接指定的IP和端口的redis实例。它的返回值是redisContext类型的。这个结构封装了一些客户端与服务端之间的连接状态，obuf是用来存放返回结果的缓冲区，同时还有客户端与服务端的协议。

``` c
//hiredis.h
/* Context for a connection to Redis */
typedef struct redisContext {
    int err; /* Error flags, 0 when there is no error */
    char errstr[128]; /* String representation of error when applicable */
    int fd;
    int flags;
    char *obuf; /* Write buffer */
    redisReader *reader; /* Protocol reader */

    enum redisConnectionType connection_type;
    struct timeval *timeout;

    struct {
        char *host;
        char *source_addr;
        int port;
    } tcp;

    struct {
        char *path;
    } unix_sock;

} redisContext;
```

redisConnect的实现比较简单，首先初始化一个redisContext变量，然后把客户端的flags字段设置为阻塞状态，接着调用redisContextConnectTcp命令。

``` c
redisContext *redisConnect(const char *ip, int port) {
    redisContext *c;

    c = redisContextInit();
    if (c == NULL)
        return NULL;

    c->flags |= REDIS_BLOCK;
    redisContextConnectTcp(c,ip,port,NULL);
    return c;
}
```

##### redisContextConnectTcp

redisContextConnectTcp()函数在net.c文件中，它调用的是_redisContextConnectTcp()这个函数，所以我们主要关注这个函数。它用来与服务端创建TCP连接，首先调整了tcp的host和timeout字段，然后getaddrinfo获取要连接的服务信息，这里兼容了IPv6和IPv4。然后尝试连接服务端。

``` c
if (connect(s,p->ai_addr,p->ai_addrlen) == -1) {
    if (errno == EHOSTUNREACH) {
        redisContextCloseFd(c);
        continue;
    } else if (errno == EINPROGRESS && !blocking) {
        /* This is ok. */
    } else if (errno == EADDRNOTAVAIL && reuseaddr) {
        if (++reuses >= REDIS_CONNECT_RETRIES) {
            goto error;
        } else {
            redisContextCloseFd(c);
            goto addrretry;
        }
    } else {
        if (redisContextWaitReady(c,timeout_msec) != REDIS_OK)
            goto error;
    }
}
```

connect()函数用于去连接服务器，连接上之后，服务器端会调用accept函数。如果连接失败，也会根据情况决定是否要关闭redisContext文件描述符。

#### 发送命令并接收返回

当客户端和服务端建立连接之后，客户端向服务器端发送命令并接收返回值了。

##### repl

我们回到redis-cli.c文件中的repl()函数，这个函数就是用来向服务器端发送命令并且接收到的结果返回。

这里首先调用了cliInitHelp()和cliIntegrateHelp()这两个函数，初始化了一些帮助信息，然后设置了一些回调的方法。如果是终端模式，则会从rc文件中加载历史命令。然后调用linenoise()函数读取用户输入的命令，并以空格分隔参数。

``` c
nread = read(l.ifd,&c,1);
```

接下来是判断是否需要过滤掉重复的参数。

##### issueCommandRepeat

生成好命令后，就调用issueCommandRepeat()函数开始执行命令。

``` c
static int issueCommandRepeat(int argc, char **argv, long repeat) {
    while (1) {
        config.cluster_reissue_command = 0;
        if (cliSendCommand(argc,argv,repeat) != REDIS_OK) {
            cliConnect(CC_FORCE);

            /* If we still cannot send the command print error.
             * We'll try to reconnect the next time. */
            if (cliSendCommand(argc,argv,repeat) != REDIS_OK) {
                cliPrintContextError();
                return REDIS_ERR;
            }
         }
         /* Issue the command again if we got redirected in cluster mode */
         if (config.cluster_mode && config.cluster_reissue_command) {
            cliConnect(CC_FORCE);
         } else {
             break;
        }
    }
    return REDIS_OK;
}
```

这个函数会调用cliSendCommand()函数，将命令发送给服务器端，如果发送失败，会强制重连一次，然后再次发送命令。

##### redisAppendCommandArgv

cliSendCommand()函数又会调用redisAppendCommandArgv()函数（在hiredis.c文件中）这个函数是按照[Redis协议](https://redis.io/topics/protocol)将命令进行编码。

##### cliReadReply

然后调用cliReadReply()函数，接收服务器端返回的结果，调用cliFormatReplyRaw()函数将结果进行编码并返回。

#### 举个栗子

我们以GET命令为例，具体描述一下，从客户端到服务端，程序是如何运行的。

我们用gdb调试redis-server，将断点设置到readQueryFromClient函数这里。

``` bash
gdb src/redis-server 
GNU gdb (Ubuntu 8.1-0ubuntu3) 8.1.0.20180409-git
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from src/redis-server...done.
(gdb) b readQueryFromClient
Breakpoint 1 at 0x43c520: file networking.c, line 1379.
(gdb) run redis.conf
```

然后再调试redis-cli，断点设置cliReadReply函数。

``` bash
gdb src/redis-cli 
GNU gdb (Ubuntu 8.1-0ubuntu3) 8.1.0.20180409-git
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from src/redis-cli...done.
(gdb) b cliReadReply
Breakpoint 1 at 0x40ffa0: file redis-cli.c, line 845.
(gdb) run
```

在客户端输入get命令，发现程序在断点处停止。

``` bash
127.0.0.1:6379> get jackey

Breakpoint 1, cliReadReply (output_raw_strings=output_raw_strings@entry=0)
    at redis-cli.c:845
845	static int cliReadReply(int output_raw_strings) {
```

我们可以看到这时Redis已经准备好将命令发送给服务端了，先来查看一下要发送的内容。

``` bash
(gdb) p context->obuf
$1 = 0x684963 "*2\r\n$3\r\nget\r\n$6\r\njackey\r\n"
```

把\r\n替换成换行符看的后是这样：

``` bash
*2
$3
get
$6
jackey

```

*2表示命令参数的总数，包括命令的名字，也就是告诉服务端应该处理两个参数。

$3表示第一个参数的长度。

get是命令名，也就是第一个参数。

$6表示第二个参数的长度。

jackey是第二个参数。

当程序运行到redisGetReply时就会把命令发送给服务端了，这时我们再来看服务端的运行情况。

``` bash
Thread 1 "redis-server" hit Breakpoint 1, readQueryFromClient (
    el=0x7ffff6a41050, fd=7, privdata=0x7ffff6b1e340, mask=1)
    at networking.c:1379
1379	void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
(gdb) 
```

程序调整到

``` bash
sdsIncrLen(c->querybuf,nread);
```

这时nread的内容会被加到c->querybuf中，我们来看一下是不是我们发送过来的命令。

``` bash
(gdb) p c->querybuf
$1 = (sds) 0x7ffff6a75cc5 "*2\r\n$3\r\nget\r\n$6\r\njackey\r\n"
```

到这里，Redis的服务端已经接受到请求了。接下来就是处理命令的过程，前文我们提到Redis是在processCommand()函数中处理的。

processCommand()函数会调用lookupCommand()函数，从redisCommandTable表中查询出要执行的函数。然后调用c->cmd->proc(c)执行这个函数，这里我们get命令对应的是getCommand函数，getCommand里只是调用了getGenericCommand()函数。

``` c
//t_string.c
int getGenericCommand(client *c) {
    robj *o;

    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.null[c->resp])) == NULL)
        return C_OK;

    if (o->type != OBJ_STRING) {
        addReply(c,shared.wrongtypeerr);
        return C_ERR;
    } else {
        addReplyBulk(c,o);
        return C_OK;
    }
}
```

lookupKeyReadOrReply()用来查找指定key存储的内容。并返回一个Redis对象，它的实现在db.c文件中。

``` c
robj *lookupKeyReadOrReply(client *c, robj *key, robj *reply) {
    robj *o = lookupKeyRead(c->db, key);
    if (!o) addReply(c,reply);
    return o;
}
```

在lookupKeyReadWithFlags函数中，会先判断这个key是否过期，如果没有过期，则会继续调用lookupKey()函数进行查找。

``` c
robj *lookupKey(redisDb *db, robj *key, int flags) {
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);

        /* Update the access time for the ageing algorithm.
         * Don't do it if we have a saving child, as this will trigger
         * a copy on write madness. */
        if (server.rdb_child_pid == -1 &&
            server.aof_child_pid == -1 &&
            !(flags & LOOKUP_NOTOUCH))
        {
            if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
                updateLFU(val);
            } else {
                val->lru = LRU_CLOCK();
            }
        }
        return val;
    } else {
        return NULL;
    }
}
```

在这个函数中，先调用了dictFind函数，找到key对应的entry，然后再从entry中取出val。

找到val后，我们回到getGenericCommand函数中，它会调用addReplyBulk函数，将返回值添加到client结构的buf字段。

``` bash
(gdb) p c->buf
$18 = "$3\r\nzhe\r\n\n$8\r\nflushall\r\n:-1\r\n", '\000' <repeats 16354 times>
```

到这里，get命令的处理过程已经完结了，剩下的事情就是将结果返回给客户端，并且等待下次命令。

客户端收到返回值后，如果是控制台输出，则会调用cliFormatReplyTTY对结果进行解析

``` bash
(gdb) n
912	                out = cliFormatReplyTTY(reply,"");
(gdb) n
918	        fwrite(out,sdslen(out),1,stdout);
(gdb) p out
$5 = (sds) 0x6949b3 "\"zhe\"\n"
```

最后将结果输出。

#### 推荐阅读

[走近源码：Redis如何执行命令](https://jackeyzhe.github.io/2019/01/05/%E8%B5%B0%E8%BF%91%E6%BA%90%E7%A0%81%EF%BC%9ARedis%E5%A6%82%E4%BD%95%E6%89%A7%E8%A1%8C%E5%91%BD%E4%BB%A4/#more)

[More Redis internals: Tracing a GET & SET](https://pauladamsmith.com/blog/2011/03/redis_get_set.html)

[GDB cheatsheet ](https://darkdust.net/files/GDB%20Cheat%20Sheet.pdf)

