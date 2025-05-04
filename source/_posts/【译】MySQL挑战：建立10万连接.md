---
title: 【译】MySQL挑战：建立10万连接
date: 2019-03-07 22:16:08
tags: MySQL
comments: true
---

原文地址：https://www.percona.com/blog/2019/02/25/mysql-challenge-100k-connections/

本文的目的是探索一种在一台MySQL服务器上建立10w个连接的方法。我们要建立的是可以执行查询的连接，而不是10w个空闲连接。<!-- more -->

你可能会问，我的MySQL服务器真的需要10w连接吗？我见过很多不同的部署方案，例如使用连接池，每个应用的连接池里放1000个连接，部署100个这样的应用服务器。还有一些非常糟糕的实践，使用“查询慢则重连并重试”的技术。这会造成雪球效应，有可能导致在几秒内需要建立上千个连接的情况。

所以我决定设置一个“小目标”，看能否实现。

#### 准备阶段

先看一下硬件，服务器由packet.net（一个云服务商）提供，配置如下：

instance size: c2.medium.x86
Physical Cores @ 2.2 GHz
(1 X AMD EPYC 7401P)
Memory: 64 GB of ECC RAM
Storage : INTEL® SSD DC S4500, 480GB

我们需要5台这样的服务器，1台用来作MySQL服务器，其余4台作为客户端。MySQL服务器使用的是Percona  Server的带有线程池插件的MySQL 8.0.13-4，这个插件需要支持上千个连接。

#### 初始化服务器配置

网络设置：

``` shell
- { name: 'net.core.somaxconn', value: 32768 }
- { name: 'net.core.rmem_max', value: 134217728 }
- { name: 'net.core.wmem_max', value: 134217728 }
- { name: 'net.ipv4.tcp_rmem', value: '4096 87380 134217728' }
- { name: 'net.ipv4.tcp_wmem', value: '4096 87380 134217728' }
- { name: 'net.core.netdev_max_backlog', value: 300000 }
- { name: 'net.ipv4.tcp_moderate_rcvbuf', value: 1 }
- { name: 'net.ipv4.tcp_no_metrics_save', value: 1 }
- { name: 'net.ipv4.tcp_congestion_control', value: 'htcp' }
- { name: 'net.ipv4.tcp_mtu_probing', value: 1 }
- { name: 'net.ipv4.tcp_timestamps', value: 0 }
- { name: 'net.ipv4.tcp_sack', value: 0 }
- { name: 'net.ipv4.tcp_syncookies', value: 1 }
- { name: 'net.ipv4.tcp_max_syn_backlog', value: 4096 }
- { name: 'net.ipv4.tcp_mem', value: '50576   64768 98152' }
- { name: 'net.ipv4.ip_local_port_range', value: '4000 65000' }
- { name: 'net.ipv4.netdev_max_backlog', value: 2500 }
- { name: 'net.ipv4.tcp_tw_reuse', value: 1 }
- { name: 'net.ipv4.tcp_fin_timeout', value: 5 }
```

系统限制设置：

``` shell
[Service]
LimitNOFILE=1000000
LimitNPROC=500000
```

相应的MySQL配置（my.cnf文件）：

``` shell
back_log=3500
max_connections=110000
```

客户端使用的是sysbench0.5版本，而不是1.0.x。具体原因我们在后面做解释。

执行命令：sysbench --test=sysbench/tests/db/select.lua --mysql-host=139.178.82.47 --mysql-user=sbtest--mysql-password=sbtest --oltp-tables-count=10 --report-interval=1 --num-threads=10000 --max-time=300 --max-requests=0 --oltp-table-size=10000000 --rand-type=uniform --rand-init=on run

#### 第一步，10,000个连接

这一步非常简单，我们不需要做过多调整就可以实现。这一步只需要一台机器做客户端，不过客户端有可能会有如下错误：

`FATAL: error 2004: Can't create TCP/IP socket (24)`

这是由于打开文件数限制，这个限制限制了TCP/IP的sockets数量，可以在客户端上进行调整：

`ulimit -n100000`

此时我们来观察一下性能：

``` shell
[  26s] threads: 10000, tps: 0.00, reads: 33367.48, writes: 0.00, response time: 3681.42ms (95%), errors: 0.00, reconnects:  0.00
[  27s] threads: 10000, tps: 0.00, reads: 33289.74, writes: 0.00, response time: 3690.25ms (95%), errors: 0.00, reconnects:  0.00
```

#### 第二步，25,000个连接

这一步会在MySQL服务端发生错误：

`Can't create a new thread (errno 11); if you are not out of available memory, you can consult the manualfor a possible OS-dependent bug`

关于这个问题的解决办法可以看这个链接：

https://www.percona.com/blog/2013/02/04/cant_create_thread_errno_11/

不过这个办法不适用于我们现在的情况，因为我们已经把所有限制调到最高：

``` shell
cat /proc/`pidof mysqld`/limits
Limit                     Soft Limit Hard Limit           Units
Max cpu time              unlimited  unlimited            seconds
Max file size             unlimited  unlimited            bytes
Max data size             unlimited  unlimited            bytes
Max stack size            8388608    unlimited            bytes
Max core file size        0          unlimited            bytes
Max resident set          unlimited  unlimited            bytes
Max processes             500000     500000               processes
Max open files            1000000    1000000              files
Max locked memory         16777216   16777216             bytes
Max address space         unlimited  unlimited            bytes
Max file locks            unlimited  unlimited            locks
Max pending signals       255051     255051               signals
Max msgqueue size         819200     819200               bytes
Max nice priority         0          0
Max realtime priority     0          0
Max realtime timeout      unlimited unlimited            us
```

这也是为什么我们最开始要选择有线程池的服务：https://www.percona.com/doc/percona-server/8.0/performance/threadpool.html

在my.cnf文件中加上下面这行设置，然后重启服务

`thread_handling=pool-of-threads`

查看一下结果

``` shell
[   7s] threads: 25000, tps: 0.00, reads: 33332.57, writes: 0.00, response time: 974.56ms (95%), errors: 0.00, reconnects:  0.00
[   8s] threads: 25000, tps: 0.00, reads: 33187.01, writes: 0.00, response time: 979.24ms (95%), errors: 0.00, reconnects:  0.00
```

吞吐量相同，但是又95%的响应从3690 ms降到了979 ms。

#### 第三步，50,000个连接

到这里，我们遇到了最大的挑战。首先尝试建立5w连接的时候，sysbench报错：

`FATAL: error 2003: Can't connect to MySQL server on '139.178.82.47' (99)`

Error (99)错误比较神秘，它意味着不能分配指定的地址。这个问题是由一个应用可以打开的端口数限制引起的，我们的系统默认配置是：

`cat /proc/sys/net/ipv4/ip_local_port_range : 32768   60999`

这表示我们只有28,231可用端口（60999减32768），或者是你最多能建立的到指定IP地址的TCP连接数。你可以在服务器和客户端扩宽这个范围：

`echo 4000 65000 > /proc/sys/net/ipv4/ip_local_port_range`

这样我们就能建立61,000个连接了，这已经接近一个IP可用端口的最大限制了（65535）。这里的关键点是，如果我们想要达到10w连接，就需要为MySQL服务器分配更多的IP地址，所以我为MySQL服务器分配了两个IP地址。

解决了端口个数问题后，我们又遇到了新的问题：

``` shell
sysbench 0.5:  multi-threaded system evaluation benchmark
Running the test with following options:
Number of threads: 50000
FATAL: pthread_create() for thread #32352 failed. errno = 12 (Cannot allocate memory)
```

这个问题是由sysbench的内存分配问题引起的。sysbench只能分配的内存只能创建32,351个连接，这个问题在1.0.x版本中更为严重。

##### Sysbench 1.0.x的限制

Sysbench 1.0.x版本使用了不同的Lua编译器，导致我们不可能创建超过4000个连接。所以看起来Sysbench比 Percona Server更早到达了极限，所以我们需要使用更多的客户端。每个客户端最多32,351个连接的话，我们最少要使用4个客户端才能达到10w连接的目标。

为了达到5w连接，我们使用了两台机器做客户端，每台机器开启25,000个线程。结果如下：

``` shell
[  29s] threads: 25000, tps: 0.00, reads: 16794.09, writes: 0.00, response time: 1799.63ms (95%), errors: 0.00, reconnects:  0.00
[  30s] threads: 25000, tps: 0.00, reads: 16491.03, writes: 0.00, response time: 1800.70ms (95%), errors: 0.00, reconnects:  0.00
```

吞吐量和上一步差不多（总的tps是16794*2 = 33588），但是性能降低了，有95%的响应时间长了一倍。这是意料之中的事情，因为与上一步相比，我们的连接数扩大了一倍。

#### 第四步，75,000个连接

这一步我们再增加一台服务器做客户端，每台客户端上同样是跑25,000个线程。结果如下：

``` shell
[ 157s] threads: 25000, tps: 0.00, reads: 11633.87, writes: 0.00, response time: 2651.76ms (95%), errors: 0.00, reconnects:  0.00
[ 158s] threads: 25000, tps: 0.00, reads: 10783.09, writes: 0.00, response time: 2601.44ms (95%), errors: 0.00, reconnects:  0.00
```

#### 第五步，100,000个连接

终于到站了，这一步同样没什么困难，只需要再开一个客户端，同样跑25,000个线程。结果如下：

``` shell
[ 101s] threads: 25000, tps: 0.00, reads: 8033.83, writes: 0.00, response time: 3320.21ms (95%), errors: 0.00, reconnects:  0.00
[ 102s] threads: 25000, tps: 0.00, reads: 8065.02, writes: 0.00, response time: 3405.77ms (95%), errors: 0.00, reconnects:  0.00
```

吞吐量仍然保持在32260的水平（8065*4），95%的响应时间是3405ms。

这里有个非常重要的事情，想必大家已经发现了：在有线程的情况下10w连接数的响应速度甚至要优于没有线程池的情况下的1w连接数的响应速度。**线程池使得Percona Server可以更加有效的管理资源，然后提供更好的响应速度。**

#### 结论

10w连接数是可以实现的，并且可以更多，实现这个目标有三个重要的组件：

1. Percona Server的线程池
2. 正确的网络设置
3. 为MySQL服务器配置多个IP地址（每个IP限制65535个连接）

#### 附录

最后贴上完整的my.cnf文件

``` shell
[mysqld]
datadir {{ mysqldir }}
ssl=0
skip-log-bin
log-error=error.log
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
character_set_server=latin1
collation_server=latin1_swedish_ci
skip-character-set-client-handshake
innodb_undo_log_truncate=off
# general
table_open_cache = 200000
table_open_cache_instances=64
back_log=3500
max_connections=110000
# files
innodb_file_per_table
innodb_log_file_size=15G
innodb_log_files_in_group=2
innodb_open_files=4000
# buffers
innodb_buffer_pool_size= 40G
innodb_buffer_pool_instances=8
innodb_log_buffer_size=64M
# tune
innodb_doublewrite= 1
innodb_thread_concurrency=0
innodb_flush_log_at_trx_commit= 0
innodb_flush_method=O_DIRECT_NO_FSYNC
innodb_max_dirty_pages_pct=90
innodb_max_dirty_pages_pct_lwm=10
innodb_lru_scan_depth=2048
innodb_page_cleaners=4
join_buffer_size=256K
sort_buffer_size=256K
innodb_use_native_aio=1
innodb_stats_persistent = 1
#innodb_spin_wait_delay=96
innodb_adaptive_flushing = 1
innodb_flush_neighbors = 0
innodb_read_io_threads = 16
innodb_write_io_threads = 16
innodb_io_capacity=1500
innodb_io_capacity_max=2500
innodb_purge_threads=4
innodb_adaptive_hash_index=0
max_prepared_stmt_count=1000000
innodb_monitor_enable = '%'
performance_schema = ON
```

