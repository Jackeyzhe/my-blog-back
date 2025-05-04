---
title: 走近源码：Redis的启动过程
date: 2019-01-04 23:31:59
tags: Redis
---

当我们对不断加深对某一项技术的了解时，一定会在一个特定的时间对它的实现方式产生兴趣。没错，这就是我现在的状态，所以，多年没有读/写C语言的我，决定要啃一下Redis的源码。<!-- more -->

Redis大体上可以分为两部分：服务器和客户端（读者吐槽：你这分的也太大体了吧）。在使用时，我们先启动服务器，然后再启动客户端。由客户端向服务器发送命令，服务器处理后将结果返回给客户端。我们从“头”开始，一起来了解一下Redis服务器在启动的时候都做了哪些事情。

对于C语言来说，main函数是一个程序的的入口，Redis也不例外。Redis的main函数写在server.c文件中。由于redis启动过程相当复杂，需要判断许多条件，例如是否在集群中，或者是否是哨兵模式等等，因此我们只介绍单机redis启动过程中一些比较重要的步骤。

##### 初始化全局服务器状态

如果redis-server命令启动时使用了test参数，那么就会先进行指定的测试。接下来调用了initServerConfig()函数，这个函数初始化了一个类型为redisServer的全局变量server。redisServer这个结构包含了非常多的字段，由于篇幅限制，我们不在这里列出，如果按类别划分的话，可以分为以下类别：

- General
- Modules
- Networking
- RDB / AOF loading information
- Fast pointers to often looked up command
- Fields used only for stats
- Configuration
- AOF / RDB persistence
- Logging
- Replication
- Synchronous replication
- Limits
- Blocked clients
- Sort parameters
- Zip structure config
- time cache
- Pubsub
- Cluster
- Scripting
- Lazy free
- Latency monitor
- Assert & bug reporting
- System hardware info

如果用一句话来概括initServerConfig()函数作用，它就是用来给可以在配置文件（通常命名为redis.conf）中配置的变量初始化一个默认值。比较常用的变量有服务器端口号、日志等级等等。

##### 设置commend table

在initServerConfig()函数中，会调用populateCommandTable()函数来设置服务器的命令表，命令表的结构如下。

``` c
struct redisCommand redisCommandTable[] = {
    {"module",moduleCommand,-2,"as",0,NULL,0,0,0,0,0},
    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
    {"setnx",setnxCommand,3,"wmF",0,NULL,1,1,1,0,0},
    ...
}
```

每一项代表的含义是：

1. name：命令的名称
2. function：命令对应的函数名。redis-server处理命令时要执行的函数
3. arity：命令的参数个数，如果是-N代表大于等于N
4. sflags：命令标志，标识命令的类型（read/write/admin...）
5. flags：位掩码，由Redis根据sflags计算
6. get_keys_proc：可选函数，当下面三个项不能指定哪些参数是key时使用
7. first_key_index：第一个是key的参数
8. last_key_index：最后一个是key的参数
9. key_step：key的“步长”，比如MSET的key_step是2，因为它的参数是key,val,key,val这样的形式
10. microseconds：执行命令所需要的微秒数
11. calls：该命令被调用总次数

设置好命令表后，redis-server还会对一些常用的命令设置快速查找方式，直接赋予server的成员指针。

``` c
server.delCommand = lookupCommandByCString("del");
server.multiCommand = lookupCommandByCString("multi");
server.lpushCommand = lookupCommandByCString("lpush");
server.lpopCommand = lookupCommandByCString("lpop");
server.rpopCommand = lookupCommandByCString("rpop");
server.zpopminCommand = lookupCommandByCString("zpopmin");
server.zpopmaxCommand = lookupCommandByCString("zpopmax");
server.sremCommand = lookupCommandByCString("srem");
server.execCommand = lookupCommandByCString("exec");
server.expireCommand = lookupCommandByCString("expire");
server.pexpireCommand = lookupCommandByCString("pexpire");
server.xclaimCommand = lookupCommandByCString("xclaim");
server.xgroupCommand = lookupCommandByCString("xgroup");
```

##### 初始化哨兵模式

变量初始化以后，就会将启动命令的路径和参数保存起来，以备下次重启的时候使用。如果启动的服务是哨兵模式，那么就会调用initSentinelConfig()和initSentinel()这两个方法来初始化哨兵模式。对sentinel不了解的同学可以看[这里](https://jackeyzhe.github.io/2018/11/04/%E7%8E%A9%E8%BD%ACRedis%E9%9B%86%E7%BE%A4%E4%B9%8BSentinel/)。initSentinelConfig()和initSentinel()都在sentinel.c文件中。initSentinelConfig函数负责初始化sentinel的端口号，以及解除服务器的保护模式。initSentinel函数负责将command table设置为只支持sentinel命令，以及初始化sentinelState数据格式。

##### 修复持久化文件

启动模式如果是redis-check-rdb/aof，那么就会执行redis_check_rdb_main()或redis_check_aof_main()这两个函数来修复持久化文件，不过redis_check_rdb_main函数所做的事情在Redis启动过程中已经做了，所以这里不需要做，直接使这个函数加载错误就可以了。

##### 处理参数

如果是简单的参数例如-v或--version、-h或--help，就会直接调用相应的方法，打印信息。如果是使用其他配置文件，则修改server.exec_argv。对于其他信息，会将他们转换成字符串，然后添加进配置文件，例如“--port 6380”就会被转换成“port 6380\n”加进配置文件。这时，redis就会调用loadServerConfig()函数来加载配置文件，这个过程会覆盖掉前面初始化默认配置文件的变量的值。

##### initServer()

initServer()函数负责结束server变量初始化工作。首先设置处理信号（SIGHUP和SIGPIPE除外），接着会创建一些双向列表用来跟踪客户端、从节点等。

``` c
server.current_client = NULL;
server.clients = listCreate();
server.clients_index = raxNew();
server.clients_to_close = listCreate();
server.slaves = listCreate();
server.monitors = listCreate();
server.clients_pending_write = listCreate();
server.slaveseldb = -1; /* Force to emit the first SELECT command. */
server.unblocked_clients = listCreate();
server.ready_keys = listCreate();
server.clients_waiting_acks = listCreate();
```

##### Shared object

createSharedObjects()函数会创建一些shared对象保存在全局的shared变量中，对于不同的命令，可能会有相同的返回值（比如报错）。这样在返回时就不必每次都去新增对象了，保存到内存中了。这个设计就是以Redis启动时多消耗一些时间为代价，换取运行的更小的延迟。

``` c
shared.crlf = createObject(OBJ_STRING,sdsnew("\r\n"));
shared.ok = createObject(OBJ_STRING,sdsnew("+OK\r\n"));
shared.err = createObject(OBJ_STRING,sdsnew("-ERR\r\n"));
shared.emptybulk = createObject(OBJ_STRING,sdsnew("$0\r\n\r\n"));
shared.czero = createObject(OBJ_STRING,sdsnew(":0\r\n"));
shared.cone = createObject(OBJ_STRING,sdsnew(":1\r\n"));
shared.cnegone = createObject(OBJ_STRING,sdsnew(":-1\r\n"));
shared.nullbulk = createObject(OBJ_STRING,sdsnew("$-1\r\n"));
shared.nullmultibulk = createObject(OBJ_STRING,sdsnew("*-1\r\n"));
shared.emptymultibulk = createObject(OBJ_STRING,sdsnew("*0\r\n"));
shared.pong = createObject(OBJ_STRING,sdsnew("+PONG\r\n"));
shared.queued = createObject(OBJ_STRING,sdsnew("+QUEUED\r\n"));
shared.emptyscan = createObject(OBJ_STRING,sdsnew("*2\r\n$1\r\n0\r\n*0\r\n"));
shared.wrongtypeerr = createObject(OBJ_STRING,sdsnew(
    "-WRONGTYPE Operation against a key holding the wrong kind of value\r\n"));
shared.nokeyerr = createObject(OBJ_STRING,sdsnew(
    "-ERR no such key\r\n"));
```

##### Shared integers

除了上述的一些返回值以外，createSharedObjects()函数还会创建一些共享的整数对象。对Redis来说，有许多类型（比如lists或者sets）都需要一些整数（比如数量），这时就可以复用这些已经创建好的整数对象，而不需要重新分配内存并创建。这同样是牺牲了启动时间来换取运行时间。

##### 新增循环事件

initServer()函数调用aeCreateEventLoop()函数(ae.c文件)来增加循环事件，并将结果返回给server的el成员。Redis使用不同的函数来兼容各个平台，在Linux平台使用epoll，在BSD使用kqueue，都不是的话，最终会使用select。Redis轮询新的连接以及I/O事件，有新的事件到来时就会及时作出响应。

##### 分配数据库

Redis初始化需要的数据库，并将结果赋给server的db成员。

``` c
server.db = zmalloc(sizeof(redisDb)*server.dbnum);
```

##### 监听TCP端口

listenToPort()用来初始化一些文件描述符，从而监听server配置的地址和端口。listenToPort函数会根据参数中的地址判断要监听的是IPv4还是IPv6，对应的调用anetTcpServer()或anetTcp6Server()函数，如果参数中未指明地址，则会强行绑定0.0.0.0

##### 初始化LRU键池

evictionPoolAlloc()（evict.c文件中）用于初始化LRU的键池，Redis的key过期策略是近似LRU算法。

``` c
void evictionPoolAlloc(void) {
    struct evictionPoolEntry *ep;
    int j;

    ep = zmalloc(sizeof(*ep)*EVPOOL_SIZE);
    for (j = 0; j < EVPOOL_SIZE; j++) {
        ep[j].idle = 0;
        ep[j].key = NULL;
        ep[j].cached = sdsnewlen(NULL,EVPOOL_CACHED_SDS_SIZE);
        ep[j].dbid = 0;
    }
    EvictionPoolLRU = ep;
}
```

##### Server cron

initServer()函数接下来会为数据库和pub/sub再生成一些列表和字典，重置一些状态，标记系统启动时间。在这之后，Redis会执行aeCreateTimeEvent()（在ae.c文件中）函数，用来新建一个循环执行serverCron()函数的事件。serverCron()默认每100毫秒执行一次。

``` c
 /* Create the timer callback, this is our way to process many background
  * operations incrementally, like clients timeout, eviction of unaccessed
  * expired keys and so forth. */
if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
    serverPanic("Can't create event loop timers.");
    exit(1);
}
```

可以看到，代码中创建循环事件时指定每毫秒执行一次serverCron()函数，这是为了使循环马上启动，但是serverCron()函数的返回值又会被作为下次执行的时间间隔。默认为1000/server.hz。server.hz随着客户端数量的增加而增加。

serverCron()函数做了许多定时执行的任务，包括rehash、后台持久化，AOF重新与清理、清理过期key，交换虚拟内存、同步主从节点等等。总之能想到的Redis的定时任务几乎都在serverCron()函数中处理。

##### 打开AOF文件

``` c
/* Open the AOF file if needed. */
if (server.aof_state == AOF_ON) {
    server.aof_fd = open(server.aof_filename,
                         O_WRONLY|O_APPEND|O_CREAT,0644);
    if (server.aof_fd == -1) {
        serverLog(LL_WARNING, "Can't open the append-only file: %s",
                  strerror(errno));
        exit(1);
    }
}
```

##### 最大内存限制

对于32位系统，最大内存是4GB，如果用户没有明确指出Redis可使用的最大内存，那么这里默认限制为3GB。

``` c
/* 32 bit instances are limited to 4GB of address space, so if there is
     * no explicit limit in the user provided configuration we set a limit
     * at 3 GB using maxmemory with 'noeviction' policy'. This avoids
     * useless crashes of the Redis instance for out of memory. */
if (server.arch_bits == 32 && server.maxmemory == 0) {
    serverLog(LL_WARNING,"Warning: 32 bit instance detected but no memory limit set. Setting 3 GB maxmemory limit with 'noeviction' policy now.");
    server.maxmemory = 3072LL*(1024*1024); /* 3 GB */
    server.maxmemory_policy = MAXMEMORY_NO_EVICTION;
}
```

##### Redis Server启动

如果Redis被设置为后台运行，此时Redis会尝试写pid文件，默认路径是/var/run/redis.pid。这时，Redis服务器已经启动，不过还有一些事情要做。

##### 从磁盘加载数据

如果存在AOF文件或者dump文件（都有的话AOF文件的优先级高），loadDataFromDisk()函数负责将数据从磁盘加载到内存。

##### 最后的设置

每次进入循环事件时，要调用beforeSleep()函数，它做了以下这些事情：

- 如果server是cluster中的一个节点，调用clusterBeforeSleep()函数
- 执行一个快速的周期
- 如果有客户端在前一个循环事件被阻塞了，向所有的从节点发送ACK请求
- 取消在同步备份过程中被阻塞的客户端的阻塞状态
- 检查是否有因为阻塞命令而被阻塞的客户端，如果有，解除
- 把AOF缓冲区写到磁盘
- 线程释放GIL

##### 进入主循环事件

程序调用aeMain()函数，进入主循环，这时其他的一些循环事件也会分别被调用

``` c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

到此，Redis服务器已经完全准备好处理各种事件了。后面我们会继续了解Redis命令执行过程究竟做了哪些事情。



`参考`：[Redis: under the hood](https://pauladamsmith.com/articles/redis-under-the-hood.html)