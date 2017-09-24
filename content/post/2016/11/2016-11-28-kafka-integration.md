---
author: wukn
catalog: true
date: 2016-11-28T05:00:00Z
tags:
- Kafka
title: '[译][Kafka]集成Kafka'
url: /2016/11/28/kafka-integration/
---

> 《Learing Apache Kafka-Second Edition》

<!--more-->

### Storm集成Kafka

#### Storm简介

少量数据的实时处理可以使用JMS（Java Messaging Service）这类技术，但是数据量很大时便会出现性能瓶颈。而且这些方案不适合横向扩展。

[Storm](http://storm.apache.org/)是开源的分布式实时数据处理系统。它可用于很多场景，如实时分析（real-time analytics）、在线机器学习（online machine learning）、连续计算（continuous computation）、数据抽取转换加载（ETL:Extract Transformation Load）。

Storm中流数据处理相关的组件：
* Spout：源源不断的数据流。
* Bolt：spout将数据传递给bolt。所有的数据处理都是由bolt完成，如数据的过滤、聚合、计算、存储等。

![](/img/post/kafka/integration/storm.png)

我们可以将Storm看成很多bolt组成的链，每个bolt对spout提供的数据流进行一些处理。

* Tuple：Storm所使用的数据结构。
* Stream：代表一个tuple序列。
* Workers：代表Storm process。
* Executers：worker发起的Storm线程。worker可以运行一个或多个executer，每个executer又可以运行一个或多个job。

#### Storm集成Kafka

集成Storm与Kafka集群需要使用[storm-kafka spout](https://github.com/apache/storm/tree/master/external/storm-kafka)。它提供了一些特性，如动态发现Kafka broker、“exactly once” tuple processing。除了常规的针对Kafka的Storm spout，它还提供了针对kafka的Trident spout实现。

> [Trident](https://storm.apache.org/documentation/Trident-tutorial.html) is a high-level abstraction for doing realtime computing on top of Storm. It allows you to seamlessly intermix high throughput (millions of messages per second), stateful stream processing with low latency distributed querying.

两种spout的实现都使用`BrokerHost`接口来跟踪Kafka broker host-to-partition 映射和`KafkaConfig`参数。`BrokerHost`接口有两个实现：`ZkHosts`和`StaticHosts`。

`ZkHosts`用于动态跟踪Kafka broker host-to-partition 映射：
* `public ZkHosts(String brokerZkStr, String brokerZkPath)`
* `public ZkHosts(String brokerZkStr)`
参数`brokerZkStr`可以是`localhost:9092`；参数`brokerZkPath`是topic和partition信息存储的根目录，默认值是`/brokers`。

`StaticHosts`用于静态分区信息：

```java
//localhost:9092. Uses default port as 9092.
Broker brokerPartition0 = new Broker("localhost");
//localhost:9092. Takes the port explicitly
Broker brokerPartition1 = new Broker("localhost", 9092);
//localhost:9092 specified as one string.
Broker brokerPartition2 = new Broker("localhost:9092");
GlobalPartitionInformation partitionInfo = new GlobalPartitionInformation();
//mapping form partition 0 to brokerPartition0
partitionInfo.addPartition(0, brokerPartition0);
//mapping form partition 1 to brokerPartition1
partitionInfo.addPartition(1, brokerPartition1);
//mapping form partition 2 to brokerPartition2
partitionInfo.addPartition(2, brokerPartition2);
StaticHosts hosts = new StaticHosts(partitionInfo);
```

创建StaticHosts实例时，首先要创建GlobalPartitionInformation实例，其次是KafkaConfig实例用来构造Kafka spout：
* `public KafkaConfig(BrokerHosts hosts, String topic)`
* `public KafkaConfig(BrokerHosts hosts, String topic, String clientId)`
参数`BrokerHosts`为Kafka broker列表；参数`topic`为topic名称；参数`clientId`被用做ZooKeeper路径的一部分，spout作为consumer在ZooKeeper中存储当前消费的offset。

`KafkaConfig`类还有一些public类型的变量，用于控制应用的行为和spout从Kafka集群获取消息的方式：

```java
public int fetchSizeBytes = 1024 * 1024;
public int socketTimeoutMs = 10000;
public int fetchMaxWait = 10000;
public int bufferSizeBytes = 1024 * 1024;
public MultiScheme scheme = new RawMultiScheme();
public boolean forceFromStart = false;
public long startOffsetTime = kafka.api.OffsetRequest.EarliestTime();
public long maxOffsetBehind = Long.MAX_VALUE;
public boolean useStartOffsetTimeIfOffsetOutOfRange = true;
public int metricsTimeBucketSizeInSecs = 60;
```

`Spoutconfig`类扩展了`KafkaConfig`类：

```java
public SpoutConfig(BrokerHosts hosts, String topic, String zkRoot, String id);
```
参数`zkRoot`为ZooKeeper的根路径；参数`id`为spout的唯一标识。

初始化`KafkaSpout`实例的代码如下：

```java
// Creating instance for BrokerHosts interface implementation
BrokerHosts hosts = new ZkHosts(brokerZkConnString);
/ Creating instance of SpoutConfig
SpoutConfig spoutConfig = new SpoutConfig(brokerHosts, topicName, "/" + topicName, UUID.randomUUID().toString());
// Defines how the byte[] consumed from kafka gets transformed into a storm tuple
spoutConfig.scheme = new SchemeAsMultiScheme(new StringScheme());
// Creating instance of KafkaSpout
KafkaSpout kafkaSpout = new KafkaSpout(spoutConfig);
```

![](/img/post/kafka/integration/storm-integrate-kafka.png)

Kafka spout与Storm使用同一个ZooKeeper实例，来存储offset的状态和记录已经消费的segment。这些offset被存储在ZooKeeper指定的根路径下。kafka spout在下游故障或超时时使用这些offset来重新处理tuple。spout可以倒回到之前的offset而不是从最后保存的offset开始，Kafka根据指定的时间戳来选择offset：

```java
spoutConfig.forceStartOffsetTime(TIMESTAMP);
```
这里，值`-1`强制spout从最新的offset重启；值`-2`强制spout从最早的offset重启。

### Hadoop集成Kafka

资源共享、稳定性、可用性、可伸缩性是分布式计算的挑战。现如今有多了一个：TB或PB级数据的处理。

#### Hadoop简介

Hadoop是个大规模分布式批处理框架，通过很多节点并行处理数据。

Hadoop基于MapReduce框架，MapReduce提供了并行分布式大规模计算借口。Hadoop有它自己的分布式文件系统HDFS（Hadoop Distributed File System）。在典型的Hadoop集群中，HDFS将数据分成很小的块（称为block）分布到所有的节点中，同时也会为每个block建立副本以确保有节点失效时仍能从其他节点读取到数据。

![](/img/post/kafka/integration/hadoop.png)

Hadoop有以下组件：
* Name Node：这是一个与HDFS交互的单点。name node中存储数据block在节点中的分布信息。
* Second Name Node：该节点存储日志，在name node故障时使用这些日志将HDFS恢复到最后更新的状态。
* Data Node：这些节点存储由name node分配的数据block，以及其他节点中数据的副本。
* Job Tracker：负责将MapReduce job分割成更小的task。
* Task Tracker：负责执行job tracker分割的task。

data node和task tracker共享同一台机器，task的执行需要name node提供数据存储位置信息。

Hadoop集群有三种：
* 本地模式（local mode）
* 伪分布式模式（pseudo distributed mode）
* 完全分布式模式（fully distributed mode）

本地模式和伪分布式模式工作于单节点集群。本地模式中所有的Hadoop主要组件运行在同一个JVM实例中；而伪分布式模式中每个组件运行在一个单独的JVM实例中。伪分布式模式主要用于开发环境。完全分布式模式中则是每个组件运行在单独的节点中。

伪分布式集群搭建步骤如下：

1. 安装和配置JDK
2. 下载[Hadoop Common](http://mirrors.hust.edu.cn/apache/hadoop/common/)
3. 解压缩后，bin文件夹添加到`PATH`中

```bash
#Assuming your installation directory is /opt/Hadoop
export HADOOP_HOME=/opt/hadoop
export PATH=$PATH:$HADOOP HOME/bin
```
4. 配置文件`etc/hadoop/core-site.xml`：
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```
配置文件`etc/hadoop/hdfs-site.xml`：

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```
5. ssh无密码连接到localhost：

```bash
ssh localhost
```
如果ssh不能无密码连接localhost，执行以下命令：

```bash
ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
```
6. 格式化文件系统

```bash
bin/hdfs namenode -format
```
7. 启动守护进程NameNode和DataNode：

```bash
sbin/start-dfs.sh
```
8. Hadoop集群设置完毕，所在Web浏览器中通过`http://localhost:50070/`访问NameNode。

#### Hadoop集成Kafka


Kafka的源码的contrib文件夹中包含Hadoop producer和consumer的示例。

##### Hadoop producer

Hadoop producer提供了从Hadoop集群向Kafka发布数据的桥梁：

![](/img/post/kafka/integration/hadoop-integrate-kafka-producer.png)

Kafka中topic可看作URI，连接到指定Kafka broker的URI格式为：

```
kafka://<kafka-broker>/<kafka-topic>
```

Hadoop consumer有两种从Hadoop获取数据的方式：

* 使用Pig脚本和Avro格式的消息：这种方式中，Kafka producer使用Pig脚本将数据写成二进制Avro格式，每一行表示一个消息。类`AvroKafkaStorage`（Pig类`StoreFunc`的扩展）接受参数Avro Schema并连接到Kafka URI，将数据推送到Kafka集群。使用AvroKafkaStorage producer时可以很容易地在同一个Pig脚本的job中写多个topic和broker。Pig脚本示例如下：

```
REGISTER hadoop-producer_2.8.0-0.8.0.jar;
REGISTER avro-1.4.0.jar;
REGISTER piggybank.jar;
REGISTER kafka-0.8.0.jar;
REGISTER jackson-core-asl-1.5.5.jar;
REGISTER jackson-mapper-asl-1.5.5.jar;
REGISTER scala-library.jar;
member_info = LOAD 'member_info.tsv' AS (member_id : int, name : chararray);
names = FOREACH member_info GENERATE name;
STORE member_info INTO 'kafka://localhost:9092/member_info' USING kafka.bridge.AvroKafkaStorage('"string"');
```

* 使用Kafka OutputFormat类：Kafka OutputFormat类是Hadoop的OutputFormat类的扩展。这种方式消息以字节形式发布。Kafka OutputFormat类使用KafkaRecordWriter类（Hadoop类RecordWriter的扩展）将记录（也就是消息）写入Hadoop集群。

要想在job中配置producer的参数，在参数前添加前缀`kafka.output`即可。例如配置压缩格式则使用`kafka.output.compression.codec`。除此之外，Kafka broker信息（kafka.metadata.broker.list）、topic（kafka.output.topic）、schema（kafka.output.schema）被注入到job的配置中。

##### Hadoop consumer

Hadoop consumer是一个从Kafka broker拉取数据并推送到HDFS中的Hadoop job。

![](/img/post/kafka/integration/hadoop-integrate-kafka-consumer.png)

一个Hadoop job并行地将Kafka数据写到HDFS中，加载数据的mapper数量取决于输入文件夹中文件的数量。输出文件夹中包括来自Kafka的数据和更新的topic offset。每个mapper在map task结束时将最后消费的消息offset写入HDFS。如果job是被或者被重启，每个mapper只是从HDFS中存储的offset处开始读取。

Kafka的源码的contrib文件夹中包含hadoop-consumer示例。运行示例需要配置一下文件test/test.properties中的参数：
* kafka.etl.topic：要获取的topic。
* kafka.server.uri：Kafka服务器地址。
* input：输入文件夹，文件夹内有使用DataGenerator生成的topic offset。文件夹内文件的数量决定了Hadoop mapper的数量。
* output：输出文件夹，文件夹内为来自Kafka的数据和更新的topic offset。
* kafka.request.limit：用于限制取数据事件的数量。

在consumer中，实例KafkaETLRecordReader是与KafkaETLInputFormat相关的record reader。它从Kafka服务器中读取数据，从input指定的offset开始到最大可用offset或者指定的上限（kafka.request.limit）。KafkaETLJob包含一些辅助函数用于初始化job配置，SimpleKafkaETLJob设置job属性并提交Hadoop job。一旦job启动了，SimpleKafkaETLMapper就从Kafka数据写入ouptut指定的HDFS。

---

参考资料：

[《Learing Apache Kafka-Second Edition》]()
