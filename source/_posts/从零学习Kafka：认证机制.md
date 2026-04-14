---
title: 从零学习Kafka：认证机制
date: 2026-04-07 19:04:35
tags: Kafka
---

前面在学习配置参数时，有同学指出没有 SSL 相关的参数，那么今天就整体看一下 Kafka 的认证机制有哪些。<!-- more -->

### Kafka 认证机制

在当前版本中，安全机制已经变为生产环境的“标配”，Kafka 的安全框架分为**认证**、**鉴权**和**加密传输**三层。简单来说，认证机制就是解决“你是谁”的问题，鉴权机制则是解决“你能干什么”的问题。本文我们就一起来看一下 Kafka 的认证机制。

当前 Kafka 支持以下几类认证方式：SSL/TLS、SASL 和 OAuthBearer。

#### SSL/TLS

这是一种基于安全证书的认证方式，也是最安全的认证方式之一。在 Kafka 中，基于 SSL 的认证方式可以理解为是客户端和 Broker 端的双路认证。在 TCP 握手阶段，双方交换证书并验证合法性。这种认证方式的优点在于安全性很高，缺点在于证书的分发和维护成本很高，且对性能也会有一定的损耗。

#### SASL

SASL 是提供认证和数据安全服务的一种框架，Kafka 支持多种 SASL 认证机制。

- SASL/PLAIN：简单的用户名密码的方式，适合内部开发测试环境使用。

- SASL/SCRAM：使用动态用户名密码的方式，密码的哈希值存储 ZooKeeper 或 KRaft 的元数据中。这种机制比 PLAIN 的安全性高，同时支持使用命令创建用户，不需要重启 Broker，是大多数生产环境的首选方案。

- SASL/GSSAPI：集成企业级的 Kerberos 认证。这种机制可以支持单点登录，适合已有大数据生态的企业生产环境。

#### OAuthBearer

这是在 2.0 版本引入的比较新的认证方式，它是基于 OAuth2 认证框架的，在认证时，客户端会先从授权服务器获取 JWT，然后再携带 Token 访问 Kafka。通常适用于云原生环境的架构中。

### 认证方式的比较

了解了这些认证方式之后，我们再总体对这些认证方式进行一下对比。

| 认证方式        | 安全性 | 配置难度 | 性能损耗 | 适用场景                     |
| ----------- | --- | ---- | ---- | ------------------------ |
| SSL/TLS     | 极高  | 中    | 高    | 适用于一般测试场景                |
| SASL/PLAIN  | 低   | 极易   | 极低   | 内部开发测试环境                 |
| SASL/SCRAM  | 高   | 中    | 低    | 适用于中小公司的 Kafka 集群        |
| SASL/GSSAPI | 极高  | 极难   | 中    | 适用于本身已经实现 Kerberos 认证的场景 |
| OAuthBearer | 高   | 难    | 中    | 适用于已经支持 OAuth 2 的场景      |

### 配置参数

#### Broker 端

我们先来看一下 Broker 端的参数，关键参数分为三类，我们分别介绍一下。

首先是要定义监听器和协议映射，需要使用 `listener` 和 `listener.security.protocol.map`，这两个参数我们在[从零学习Kafka：配置参数](https://jackeyzhe.github.io/2026/01/26/%E4%BB%8E%E9%9B%B6%E5%AD%A6%E4%B9%A0Kafka%EF%BC%9A%E9%85%8D%E7%BD%AE%E5%8F%82%E6%95%B0/#Broker-%E7%AB%AF%E5%8F%82%E6%95%B0)一文中都有介绍，就不过多赘述了。

如果你使用的是 SASL 的认证方式，还需要配置 `sasl.enabled.mechanisms` 和 `sasl.mechanism.inter.broker.protocol` 这两个参数。

`sasl.enabled.mechanisms` 是用来指定 Broker 端允许的 SASL 机制，可以支持 `PLAIN`, `SCRAM-SHA-256`, `SCRAM-SHA-512`, `GSSAPI`，注意这个参数是可以配置多个值的。

`sasl.mechanism.inter.broker.protocol` 用来指定 Broker 节点之间通讯使用的认证机制。

最后我们需要配置 SSL 证书。

首先是 Broker 端的私钥路径和密码的配置参数：`ssl.keystore.location` 和 `ssl.keystore.password`，这两个参数用来证明“我是 Broker”。

接着是信任库的路径和密码：`ssl.truststore.location` 和 `ssl.truststore.password`，它们是用来验证客户端的证书的。

最后设置是否需要双向认证：`ssl.client.auth`，它表示是否要求客户端也提供证书，required 表示客户端必须提供证书，requested 客户端是否需要提供证书是可选的，none 表示不需要客户端提供证书。

#### Producer/Consumer 端

看完了 Broker 作为服务端的配置参数之后，我们再来看一下客户端的配置参数，也就是 Producer 和 Consumer 的配置参数。这里配置主要分为两部分，一是协议的一些基础配置，二是身份凭证配置。

协议的机制配置包括 `security.protocol` 和 `sasl.mechanism` 这两个参数。

`security.protocol` 用来指定和 Broker 通信使用的协议，必须与服务端配置的监听器协议相同。

`sasl.mechanism` 是客户端使用的 SASL 机制，必须是服务端参数 `sasl.enabled.mechanisms` 配置列表中的一个。

接下来是身份凭证配置，最核心的参数就是 `sasl.jaas.config`，它定义了具体的用户名和密码。

### 实战

在正文的最后一节，我们一起来对一个 Kafka 集群进行一次认证配置，我们使用的是 SASL/SCRAM 认证机制，感兴趣的同学可以一起手动配置一遍。

#### 1. 准备 JAAS 配置文件

在命令行中执行命令

```shell
cat <<EOF > config/kafka_server_jaas.conf
KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="admin-password";
};
EOF
```

#### 2. 修改服务端配置

修改配置文件 config/server.properties，我们将 EXTERNAL 定义为 SASL，表示外部业务强制加密。

```properties
# 监听协议映射
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,EXTERNAL:SASL_PLAINTEXT

# 监听端口（9092明文用于内部，9093加密用于外部业务）
listeners=PLAINTEXT://:9092,EXTERNAL://:9093,CONTROLLER://:9094

# 通讯协议设置
inter.broker.listener.name=PLAINTEXT

# 开启 SCRAM 认证机制
sasl.enabled.mechanisms=SCRAM-SHA-256

# 控制器监听器
controller.listener.names=CONTROLLER

# 仲裁投票节点地址（确保这里的端口 9094 与上面 listeners 中的 CONTROLLER 端口一致）
controller.quorum.voters=1@localhost:9094
```

#### 3. 设置环境变量并启动 Kafka

```shell
# 设置环境变量
export KAFKA_OPTS="-Djava.security.auth.login.config=$(pwd)/config/kafka_server_jaas.conf"

# 生成集群 ID (如果还没做过)
KAFKA_CLUSTER_ID=$(bin/kafka-storage.sh random-uuid)

# 格式化存储目录
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/server.properties

# 启动 Broker
bin/kafka-server-start.sh -daemon config/server.properties
```

这里核心目的是让 Kafka 知道 JAAS 文件的位置。

#### 4. 动态创建业务用户

Kafka 集群启动后，我们创建一个业务用户，这里使用的端口是 9092，因为我们还没有给管理员配置客户端证书。

```shell
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
--alter --add-config 'SCRAM-SHA-256=[password=alice-password]' \
--entity-type users --entity-name alice
```

创建完成后 Kafka 会有以下提示。

![kafka_alice](https://res.cloudinary.com/dxydgihag/image/upload/v1776152370/Blog/Kafka/5/kafka_alice.png)

#### 5. 创建客户端配置文件

```shell
cat <<EOF > config/client-alice.conf
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \\
  username="alice" \\
  password="alice-password";
EOF
```

#### 6. 测试验证

分别使用 Producer 和 Consumer 验证消息的发送和消费

Producer 端

```shell
bin/kafka-console-producer.sh --bootstrap-server localhost:9093 \
--topic test-auth \
--producer.config config/client-alice.conf
```

![auth_producer](https://res.cloudinary.com/dxydgihag/image/upload/v1776152430/Blog/Kafka/5/auth_producer.png)

Consumer端

```shell
bin/kafka-console-consumer.sh --bootstrap-server localhost:9093 \
--topic test-auth \
--from-beginning \
--consumer.config config/client-alice.conf
```

![auth_consumer](https://res.cloudinary.com/dxydgihag/image/upload/v1776152438/Blog/Kafka/5/auth_consumer.png)

至此，我们完整了一整套 SASL/SCRAM 认证机制的配置与验证。不知道你在配置过程中有没有遇到什么问题，如果有的话可以随时与我进行交流。

### 总结

本文我们先了解了 Kafka 支持的认证机制，并对它们进行了一次简单的比较。接着又学习了认证相关的配置参数。最后实际操作配置了 SASL/SCRAM 认证机制。希望通过本文的学习大家能对 Kafka 的认证机制有一个更加清晰的认识。
