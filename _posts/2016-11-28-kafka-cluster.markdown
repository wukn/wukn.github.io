---
layout:     post
title:      "[Kafka]部署Kafka集群"
subtitle:   ""
date:       2016-11-28 01:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Kafka

---

> 《Learing Apache Kafka-Second Edition》


Kafka支持多种集群方式，例如：

* 单节点单broker集群
* 单节点多broker集群
* 多节点多broker集群

一个Kafka集群主要包含以下五个组件：

* topic：topic是producer发布的消息的类别。在Kafka中，topic是分区的，每个partition内是顺序不可变的消息序列。Kafka集群维护每个topic的partition日志。partition内每个消息分配了一个被称为offset的唯一序列ID。
* broker：一个Kafka集群包含一个或多个服务器，每个服务器有一个或多个运行的被称为broker的服务进程。topic是在broker进程的上下文中创建的。
* zookeeper：Kafka使用ZooKeeper来协调broker和cunsumers。ZooKeeper允许分布式进程通过一个共享的层级数据寄存器命名空间相互协调（这些数据寄存器被称为znode），就像文件系统一样。它跟标准文件系统的区别是，每个znode可以拥有关联的数据，且数据量有限制。ZooKeeper是被设计用来存储协调数据的，如状态信息、配置信息、位置信息等。关于ZooKeeper详见[Hadoop Wiki](http://wiki.apache.org/hadoop/ZooKeeper/ProjectDescription)。
* producers：producer选择topic内的合适的partition并向其发布数据。为了负载均衡，可以基于循环的方式或者自定义方法来关联信息与topic partition。
* consumer：consumer是订阅topics并处理发布的消息的应用或进程。

### 单节点单broker集群

在上篇中，我们在单台机器上部署了Kafka，现在将其设置为单节点单broker集群。架构如图所示：

![](/img/post/kafka/cluster/single-node-single-broker-cluster.png)

#### 启动ZooKeeper服务：

Kafka提供了一个默认的简单的ZooKeeper配置文件，可启动本地的单ZooKeeper实例。

```bash
bin/zookeeper-server-start.sh config/zookeeper.properties &
```

zookeeper.properties配置文件中定义了以下的关键属性：

```bash
# Data directory where the zookeeper snapshot is stored.
dataDir=/tmp/zookeeper
# The port listening for client request
clientPort=2181
# disable the per-ip limit on the number of connections since this is a non-production config
maxClientCnxns=0
```

在默认的配置中，ZooKeeper监听2181端口。关于构建多ZooKeeper服务，详见[http://zookeeper.apache.org/](http://zookeeper.apache.org/).

#### 启动Kafka broker

```bash
bin/kafka-server-start.sh config/server.properties &
```

server.properties配置文件中定义了以下的关键属性：

```bash
# The id of the broker. This must  be set to a unique integer for each broker.
Broker.id=0
# The port the socket server listens on
port=9092
# The directory under which to store log files
log.dir=/tmp/kafka8-logs
# The default number of log partitions per topic.  
num.partitions=2
# Zookeeper connection string  
zookeeper.connect=localhost:2181
```

#### 创建一个Kafka topic

创建一个名为`kafkatopic`的topic，单partitoin且只有一个副本：

```bash
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic kafkatopic
```

查看topic列表：

```bash
bin/kafka-topics.sh --list --zookeeper localhost:2181
```

#### 启动producer发送信息

Kafka提供了一个命令行producer，可接收命令行输入并将其作为消息发布到Kafka集群。默认每一行是一个消息。

```bash
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic kafkatopic
```
其中，broker-list和topic这两个参数是必须的，broker-list指定要连接的broker，格式为`node_address:port`。topic是必须的，因为需要发送消息给订阅了该topic的consumer group。

现在可以在命令行里输入一些信息，每一行会被作为一个消息。

默认的属性配置在/config/producer.properties文件中：

```bash
# list of brokers used for bootstrapping knowledge about the rest of the format: host1:port1,host2:port2...
metadata.broker.list=localhost:9092
# specify the compression codec for all data generated: none , gzip, snappy.
compression.codec=none
```

#### 启动consumer接收消息

```bash
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic kafkatopic --from-beginning
```

consumer加载完成后，会输出刚才在producer里输入消息。

默认的属性配置在/config/consumer.properties文件中：

```bash
# consumer group id (A string that uniquely identifies a set of consumers within the same consumer group)
group.id=test-consumer-group
```

在不同的终端里分别启动zookeeper，broker，producer，consumer后，在producer终端里输入消息，消息就会在consumer终端中显示了。

### 单节点多broker集群

接下来构建一个单节点多broker集群：

![](/img/post/kafka/cluster/single-node-multi-broker-cluster.png)

#### 启动ZooKeeper

同上。

```bash
bin/zookeeper-server-start.sh config/zookeeper.properties &
```

#### 启动broker

在单节点上配置多个bbroker时，需要为每个broker指定单独的属性配置文件，其中`broker.id`、`port`、`log.dir`这三个属性必须时不同的。

在broker1的配置文件server-1.properties中，定义参数为：

```bash
broker.id=1
port=9093
log.dir=/tmp/kafka-logs-1
```

在broker2的配置文件server-2.properties中，定义参数为：

```bash
broker.id=2
port=9094
log.dir=/tmp/kafka-logs-2
```

我们为每个broker指定了不同的端口，是因为我们在同一台机器上搭建多个broker，实际生产环境中，broker可能分布在多台机器上。

启动每个broker：

```bash
bin/kafka-server-start.sh config/server-1.properties &
bin/kafka-server-start.sh config/server-2.properties &
```

#### 创建topic

创建一个名为`replicated-kafkatopic`的topic：

```bash
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic replicated-kafkatopic
```

#### 启动producer

如果一个producer连接多个broker，需要传递参数`broker-list`：

```bash
bin/kafka-console-producer.sh --broker-list localhost:9093, localhost:9094 --topic replicated-kafkatopic
```

多个producer时，为每个指定`broker-list`。

#### 启动consumer

```bash
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic replicated-kafkatopic
```

### 多节点多broker集群

在多节点多broker集群中，每个节点都需要安装Kafka，且所有的broker都连接到同一个ZooKeeper。

![](/img/post/kafka/cluster/multi-node-multi-broker-cluster.png)

---

参考资料：

[《Learing Apache Kafka-Second Edition》]()
