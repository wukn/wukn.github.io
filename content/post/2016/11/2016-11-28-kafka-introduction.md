---
author: wukn
catalog: true
date: 2016-11-28T00:00:00Z
tags:
- Kafka
title: '[译][Kafka]Kafka简介'
url: /2016/11/28/kafka-introduction/
---

> 《Learing Apache Kafka-Second Edition》

在当前的大数据时代，第一个挑战是海量数据的收集，另一个就是这些数据的分析。数据分析的类型通常有用户行为数据、应用性能跟踪数据、活动数据日志、事件消息等。消息发布机制用于连接各种应用并在它们之间路由消息，例如通过message broker。Kafka是快速地将海量信息实时路由到消费者的解决方案，实现信息的生产者和消费者的无缝集成。它不会阻塞信息的生产者，同时信息生产者不会知道信息消费者。

<!--more-->

Apache Kafka是个开源的分布式消息发布订阅系统，具有以下特征：

* 消息持久化（persistent messaging）：Kafka提供时间复杂度为O(1)的消息持久化能力，即使是TB级别以上的数据也能保证常数时间复杂度的访问性能。在Kafka中，消息被持久化在磁盘中，同时也在集群之间复制，防止数据丢失
* 高吞吐率（high throughput）：即使在普通的机器上也能提供每秒上百MB读写的能力。
* 分布式（distributed）：Kafka支持Kafka服务器间的消息分区，以及基于消费者机器集群的分布式消费，维护每个分区内的顺序。Kafka集群可以在不停机的情况下弹性地增减结点。
* 实时（real time）：生产者线程产生的消息应该立即被消费者线程可见，这是基于事件系统的关键特性，例如Complex Event Processing (CEP) system。
* 支持在线水平扩展（scale out）

Kafka提供了实时发布订阅的解决方案，克服了实时数据消费和比实时数据更大数量级的数据量增长的问题。Kafka也支持Hadoop系统中的并行数据加载。下图展示了一种典型的使用Kafka消息系统的大数据聚合分析的场景。

![](/img/post/kafka/introduction/kafka-introduction.png)

在消息生产端，有不同类型的生产者，例如：

* 生成应用日志的前端web应用
* 生成web分析日志的代理
* 生成转换日志的适配器
* 生成调用跟踪日志的服务

在消息消费端，有不同类型的消费者，例如：

* 离线消费者（offline consumer）：消费消息，将它们存储到Hadoop或传统数据仓库用于离线分析
* 接近实时的消费者（near real-time consumer）：消费消息，将它们存储到NoSQL数据库中用于接近实时的分析
* 实时消费者（real-time consumer）：例如Spark或Storm，在内存数据库中过滤信息并触发相关事件

### 使用Kafka的场景

各种形式的web活动产生大量数据，例出用户活动事件如登录、访问页面、点击链接，社交网络活动如喜欢、分享、评论，还有系统运作日志等等。由于这些数据的高吞吐量（每秒百万级的消息），通常由日志记录系统和日志聚合解决方案来处理。这样的传统方案对提供日志数据给Hadoop这样的离线分析系统是可行的，但对于需要实时处理的系统就够呛了。

互联网应用的发展趋势表明，这样的活动数据已经成为生产数据的一部分，用于实时分析，包括：

* 基于相关性的搜索
* 基于受欢迎度、同现、或这情感分析的推荐
* 向大众提供广告
* 从垃圾邮件或抓取未经授权数据分析互联网应用程序安全性
* 设备传感器发送高温报警
* 反常用户行为或应用被侵入

Kafka的目标是统一离线和在线处理，通过提供Hadoop系统中并行加载机制和使用集群将实时消费分区的能力。Kafka可以跟Scribe或Flume相比较，因为它可以用于处理活动流数据，但是从架构的角度，它更接近于传统的消息系统，如ActiveMQ或RabitMQ。

### 使用Kafka的案例

Kafka通常被用于：

* 日志聚合（log aggregation）：从服务器收集日志文件，将它们放到文件服务器或HDFS中处理。使用Kafka可以将日志数据或事件数据抽象为信息流，从而避免了对文件格式等的依赖。同时提供了低延迟处理能力，并支持多数据源和分布式处理。
* 流处理（stream processing）：Kafka可用于对数据进行多阶段处理的场景，例如，一个主题的原始数据被消费，经过增强或转换处理形成新的主题供后续的消费者使用。这样的处理过程称为流处理。
* 提交日志（commit logs）：Kafka可用来处理大规模分布式系统的外部提交日志。在Kafka集群之间复制日志可以帮助故障节点恢复其状态。
* 点击流跟踪（click stream tracking）：另一个常用Kafka的场景是，捕捉用户点击流数据，如页面视图、搜索等这样的real-time publish-subscribe feed。这些数据被发布到中心主题，由于数据量巨大，每个活动类型一个主题。这些主题可以被很多类型的消费者订阅，如实时处理和监控应用。
* 消息处理（messaging）：message broker用于将数据处理和数据提供者解耦。Kafka可以作为message broker，它具有更好的吞吐量、内建分区、复制和容错能力。

一些公司将Kafka用于各自的场景：

* LinkedIn使用Kafka处理活动流数据和运营指标数据。这些数据支撑着除Hadoop离线分析系统外的多个系统，如LinkedIn news feed和LinkedIn Today。
* DataSift使用Kafka收集监控事件和实时跟踪用户消费数据。
* Twitter、1号店将Kafka作为Storm的一部分。
* Foursquare使用Kafka支撑在线和离线消息处理，将监控和基于Hadoop的离线的生产系统相集成。
* Square将Kafka当作总线使用，在各种各样的数据中心之间传递系统事件。

### Kafka的一些高级特性

* 为分区提供了复制因子。保证了所有提交的信息不会丢失，即使某个borker失效时分区中还有未消费的数据，至少有一个副本是可用的。默认生产者发送信息请求会阻塞直到信息提交到所有活跃的副本中，也可以通过配置指定生产者将信息提交到某个broker。
* 消费者采用长轮询模式（long-pulling model），并且会阻塞直到有可用的提交信息，避免了频繁轮询。

### 安装Kafka

Kafka是用Scala语言实现的，并用Gradle构建二进制包。使用Kafka之前安装JDK 1.7或更高版本。并配置环境变量JAVA_HOME。

1.下载Kaka：

```
wget http://mirrors.ustc.edu.cn/apache/kafka/0.9.0.0/kafka_2.10-0.9.0.0.tgz
```

2.解压缩：

```
tar xzf kafka_2.10-0.9.0.0.tgz
cd kafka_2.10-0.9.0.0
```

其中：

```
/bin 启动和停止命令等
/config 配置文件
/libs 类库
```

3.启动和停止

启动Zookeeper server：

```
bin/zookeeper-server-start.sh config/zookeeper.properties &
```

启动Kafka server:

```
bin/kafka-server-start.sh config/server.properties &  
```

停止Kafka server：

```
bin/kafka-server-stop.sh  
```

停止Zookeeper server:

```
bin/zookeeper-server-stop.sh
```

4.单机连通性测试：
运行producer：

```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test  
```
运行consumer：

```
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning  
```
在producer端输入字符串并回车，consumer端显示则表示成功。

5.分布式联通性测试

配置server.proprities中的hostname，kafka v0.10中为:

```
listeners=PLAINTEXT://<ip>:<port>
```
运行producer和consumer时将localhost替换为IP地址。

---

参考资料：

[《Learing Apache Kafka-Second Edition》]()
