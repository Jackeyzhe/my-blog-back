---
title: Flink源码阅读：集群启动
date: 2025-11-27 16:35:40
tags: Flink
---

前文中，我们已经了解了 Flink 的三种执行图是怎么生成的。今天继续看一下 Flink 集群是如何启动的。<!-- more -->

### 启动脚本

集群启动脚本的位置在：

```bash
flink-dist/src/main/flink-bin/bin/start-cluster.sh
```

脚本会负责启动 JobManager 和 TaskManager，我们主要关注 standalone 启动模式，具体的流程见下图。

![start-cluster](https://res.cloudinary.com/dxydgihag/image/upload/v1764258366/Blog/flink/11/start-cluster.png)

从图中可以看出 JobManager 是通过 jobmanager.sh 文件启动的，TaskManager 是通过taskmanager.sh 启动的，两者都调用了 flink-daemon.sh，通过传递不同的参数，最终运行不同的 Java 类。

```bash
case $DAEMON in
    (taskexecutor)
        CLASS_TO_RUN=org.apache.flink.runtime.taskexecutor.TaskManagerRunner
    ;;

    (zookeeper)
        CLASS_TO_RUN=org.apache.flink.runtime.zookeeper.FlinkZooKeeperQuorumPeer
    ;;

    (historyserver)
        CLASS_TO_RUN=org.apache.flink.runtime.webmonitor.history.HistoryServer
    ;;

    (standalonesession)
        CLASS_TO_RUN=org.apache.flink.runtime.entrypoint.StandaloneSessionClusterEntrypoint
    ;;

    (standalonejob)
        CLASS_TO_RUN=org.apache.flink.container.entrypoint.StandaloneApplicationClusterEntryPoint
    ;;

    (sql-gateway)
        CLASS_TO_RUN=org.apache.flink.table.gateway.SqlGateway
        SQL_GATEWAY_CLASSPATH="`findSqlGatewayJar`":"`findFlinkPythonJar`"
    ;;

    (*)
        echo "Unknown daemon '${DAEMON}'. $USAGE."
        exit 1
    ;;
esac
```

### JobManager 启动流程
