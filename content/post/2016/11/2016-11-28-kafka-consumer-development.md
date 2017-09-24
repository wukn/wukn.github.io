---
author: wukn
catalog: true
date: 2016-11-28T04:00:00Z
tags:
- Kafka
title: '[译][Kafka]开发Consumer'
url: /2016/11/28/kafka-consumer-development/
---

> 《Learing Apache Kafka-Second Edition》

<!--more-->

consumer就是接收producer发布的消息进行处理的应用。

![](/img/post/kafka/consumer-development/kafka-consumer.png)

上图描述了consumer消费消息的high-level层工作原理。consumer从broker内的topic订阅消息；然后consumer向lead broker发起请求，指定消息的offset。consumer使用这样的拉取模式，每次始终拉取它记录在日志中当前位置之后的所有消息。
在订阅时，consumer连接到任意活动的节点，请求关于topic的partition所在的leader的元数据。这样consumer直接和lead broker通信并接收消息。topic被分割为一系列有序的partition，每个partition只能被一个consumer消费。一旦被消费后，consumer将消息offset移到下一个待消费的消息。这样记录了消费的状态，同时又提供了倒回到之前的offset或重新消费partition的柔性。

### Kafka consumer API

#### high-level consumer API

如果只需要数据而不需要考虑消息offset相关的处理时，使用high-level API就够了。


接口`kafka.javaapi.consumer.ConsumerConnector`和它的实现类`kafka.javaapi.consumer.ZookeeperConsumerConnector`。该类负责consumer和ZooKeeper的所有交互。
* `createMessageStreams(Map<String,Integer>,Decoder<K>,Decoder<V>):Map<String,List<KafkaStream<K,V>>>`
* `createMessageStreams(Map<String,Integer>):Map<String,List<KafkaStream<byte[],byte[]>>>`
* `createMessageStreamsByFilter(TopicFilter,int,Decoder<K>,Decoder<V>):List<KafkaStream<K,V>>`
* `createMessageStreamsByFilter(TopicFilter,int):List<KafkaStream<byte[],byte[]>>`
* `createMessageStreamsByFilter(TopicFilter):List<KafkaStream<byte[],byte[]>>`
* `commitOffsets():void`
* `shutdown:void`

类`kafka.consumer.KafkaStream`的K和V分别指定partition key和message value的类型。

类`kafka.consumer.ConsumerConfig`封装了与ZooKeeper连接所需要的参数。


####  low-level consumer API

high-level consumer API隐藏了consumer与broker通信的细节。而low-level consumer API（也称simple consumer API）提供了consumer与broker通信的方法，它是无状态的。使用low-level consumer API时需要自己处理offset的跟踪、寻找topic和partition的lead broker、lead broker变更等。

类`kafka.javaapi.consumer.SimpleConsumer`包含的方法如下：
* `SimpleConsumer()`：构造函数。
* `SimpleConsumer(String, int, int, int, String):void`：构造函数，参数分别为lead broker、broker port、connection timeout、buffer size、client ID。
* `fetch(FetchRequest)`：该方法返回topic中的消息集。参数`kafka.api.FetchRequest`指定topic名称、partition、起始offset、最大字节数。
* `send(TopicMetadataRequest)`：该方法返回topic序列的元数据。参数`kafka.javaapi.TopicMetadataRequest`指定version ID、client ID、topic序列。
* `getOffsetsBefore(OffsetRequest)`：返回给定时间之前的有效的offset集。参数`kafka.javaapi.OffsetRequest`。
* `commitOffsets(OffsetCommitRequest)`：向topic提交offset。参数`kafka.javaapi.OffsetFetchRequest`指定topic。
* `fetchOffsets(OffsetFetchRequest)`：返回topic中的offset。参数`kafka.javaapi.OffsetFetchRequest`指定topic。
* `close():void`；关闭与lead broker的连接。

[关于low-level API的例子看这里。](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example)

### 一个简单的Java consumer

接下来写个使用high-level API的单线程consumer。`SimpleHLConsumer`从指定的topic拉取消息进行消费（假定topic内只有一个partition）。

1.引入以下类：

```java
import kafka.consumer.ConsumerConfig;
import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;
```

2.定义属性：

```java
Properties props = new Properties();
props.put("zookeeper.connect", "localhost:2181");
props.put("group.id", "testgroup");
props.put("zookeeper.session.timeout.ms", "500");
props.put("zookeeper.sync.time.ms", "250");
props.put("auto.commit.interval.ms", "1000");
ConsumerConfig config = new ConsumerConfig(props);
```

看一下代码中提到的属性：
* `zookeeper.connect`：
* `group.id`：
* `zookeeper.session.timeout.ms`：
* `zookeeper.sync.time.ms`：
* `auto.commit.interval.ms`：

3.从topic中读取消息：

```java
Map<String, Integer> topicMap = new HashMap<String, Integer>();
// 1 represents the single thread
topicCount.put(topic, new Integer(1));
Map<String, List<KafkaStream<byte[], byte[]>>> consumerStreamsMap = consumer.createMessageStreams(topicMap);
// Get the list of message streams for each topic, using the default decoder.
List<KafkaStream<byte[], byte[]>>streamList = consumerStreamsMap.get(topic);
for (final KafkaStream <byte[], byte[]> stream : streamList) {
    ConsumerIterator<byte[], byte[]> consumerIte = stream.iterator();
    while (consumerIte.hasNext())
        System.out.println("Message from Single Topic :: " + new String(consumerIte.next().message()));
}
```

完整代码如下：

```java
package kafka.examples.consumer;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import kafka.consumer.ConsumerConfig;
import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;
public class SimpleHLConsumer {
     private final ConsumerConnector consumer;
     private final String topic;
     public SimpleHLConsumer(String zookeeper, String groupId, String topic) {
         consumer = kafka.consumer.Consumer.createJavaConsumerConnector(createConsumerConfig(zookeeper, groupId));
         this.topic = topic;
     }
     private static ConsumerConfig createConsumerConfig(String zookeeper, String groupId) {
         Properties props = new Properties();
         props.put("zookeeper.connect", zookeeper);
         props.put("group.id", groupId);
         props.put("zookeeper.session.timeout.ms", "500");
         props.put("zookeeper.sync.time.ms", "250");
         props.put("auto.commit.interval.ms", "1000");
         return new ConsumerConfig(props);
     }
     public void testConsumer() {
         Map<String, Integer> topicMap = new HashMap<String, Integer>();
         // Define single thread for topic
         topicMap.put(topic, new Integer(1));
         Map<String, List<KafkaStream<byte[], byte[]>>> consumerStreamsMap = consumer.createMessageStreams(topicMap);
         List<KafkaStream<byte[], byte[]>> streamList = consumerStreamsMap.get(topic);
         for (final KafkaStream<byte[], byte[]> stream : streamList) {
              ConsumerIterator<byte[], byte[]> consumerIte = stream.iterator();
              while (consumerIte.hasNext())
                  System.out.println("Message from Single Topic :: "     + new String(consumerIte.next().message()));
          }
          if (consumer != null)
              consumer.shutdown();
     }
     public static void main(String[] args) {
         String zooKeeper = args[0];
         String groupId = args[1];
         String topic = args[2];
         SimpleHLConsumer simpleHLConsumer = new SimpleHLConsumer(zooKeeper, groupId, topic);
         simpleHLConsumer.testConsumer();
    }
}
```

在运行上面的代码之前，确保已经创建了名为`kafkatopic`的topic：

```bash
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 3 --topic kafkatopic
```

添加环境变量`KAFKA_LIB`指向Kafka的lib文件夹路径，并将lib文件夹下的jar包添加到`classpath`。

运行上一章的SimpleProducer：

```bash
java kafka.examples.ch4.SimpleProducer kafkatopic 100
```

编译SimpleHLConsumer：

```bash
javac -d . kafka/examples/consumer/SimpleHLConsumer.java
```

运行SimpleHLConsumer，三个参数分别为ZooKeeper连接字符串、唯一的group ID、topic名称：

```bash
java kafka.examples.consumer.SimpleHLConsumer localhost:2181 testgroup kafkatopic
```

### 多线程Java consumer

上个例子是个很简单的从单broker且topic只有一个分区消费消息的场景。接下来考虑多topic且有多个分区的场景。
通常topic内有几个partition就使用几个线程，这样可以简化线程与partition之间的关系，避免一些冲突。

1.引入以下类：

```java
import kafka.consumer.ConsumerConfig;
import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;
```

2.定义属性：

```java
Properties props = new Properties();
props.put("zookeeper.connect", "localhost:2181");
props.put("group.id", "testgroup");
props.put("zookeeper.session.timeout.ms", "500");
props.put("zookeeper.sync.time.ms", "250");
props.put("auto.commit.interval.ms", "1000");
ConsumerConfig config = new ConsumerConfig(props);
```

3.从线程读取消息

首先创建一个线程池：

```java
// Define thread count for each topic
topicMap.put(topic, new Integer(threadCount));
// Here we have used a single topic but we can also add multiple topics to topicCount MAP
Map<String, List<KafkaStream<byte[], byte[]>>> consumerStreamsMap = consumer.createMessageStreams(topicMap);
List<KafkaStream<byte[], byte[]>> streamList = consumerStreamsMap.get(topic);
// Launching the thread pool
executor = Executors.newFixedThreadPool(threadCount);
```

完整代码如下：

```java
package kafka.examples.consumer;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import kafka.consumer.ConsumerConfig;
import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;
public class MultiThreadHLConsumer {
    private ExecutorService executor;
    private final ConsumerConnector consumer;
    private final String topic;
    public MultiThreadHLConsumer(String zookeeper, String groupId, String topic) {
        consumer = kafka.consumer.Consumer.createJavaConsumerConnector(createConsumerConfig(zookeeper, groupId));
        this.topic = topic;
    }
    private static ConsumerConfig createConsumerConfig(String zookeeper, String groupId) {
        Properties props = new Properties();
        props.put("zookeeper.connect", zookeeper);
        props.put("group.id", groupId);
        props.put("zookeeper.session.timeout.ms", "500");
        props.put("zookeeper.sync.time.ms", "250");
        props.put("auto.commit.interval.ms", "1000");
        return new ConsumerConfig(props);
    }
    public void shutdown() {
        if (consumer != null)
            consumer.shutdown();
        if (executor != null)
            executor.shutdown();
    }
    public void testMultiThreadConsumer(int threadCount) {
        Map<String, Integer> topicMap = new HashMap<String, Integer>();
        // Define thread count for each topic
        topicMap.put(topic, new Integer(threadCount));
        // Here we have used a single topic but we can also add multiple topics to topicCount MAP
        Map<String, List<KafkaStream<byte[], byte[]>>> consumerStreamsMap = consumer.createMessageStreams(topicMap);
        List<KafkaStream<byte[], byte[]>> streamList = consumerStreamsMap.get(topic);
        // Launching the thread pool
        executor = Executors.newFixedThreadPool(threadCount);
        // Creating an object messages consumption
        int count = 0;
        for (final KafkaStream<byte[], byte[]> stream : streamList) {
            final int threadNumber = count;
            executor.submit(new Runnable() {
                public void run() {
                    ConsumerIterator<byte[], byte[]> consumerIte = stream.iterator();
                    while (consumerIte.hasNext()) {
                      System.out.println("Thread Number " + threadNumber + ": " + new String(consumerIte.next().message()));
                      System.out.println("Shutting down Thread Number: " + threadNumber);
                    }
                }
            });
            count++;
      }
        if (consumer != null)
            consumer.shutdown();
        if (executor != null)
            executor.shutdown();
    }
    public static void main(String[] args) {
        String zooKeeper = args[0];
        String groupId = args[1];
        String topic = args[2];
        int threadCount = Integer.parseInt(args[3]);
        MultiThreadHLConsumer multiThreadHLConsumer = new MultiThreadHLConsumer(zooKeeper, groupId, topic);
        multiThreadHLConsumer.testMultiThreadConsumer(threadCount);
        try {
            Thread.sleep(10000);
        } catch (InterruptedException ie) {
        }
        multiThreadHLConsumer.shutdown();
    }
}
```

运行代码前，先构建一个多broker集群，并创建topic：

```bash
bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic kafkatopic --partitions 4 --replication-factor 2
```

运行SimpleProducer：

```bash
java kafka.examples.producer.SimpleProducer kafkatopic 100
```

编译MultiThreadHLConsumer：

```bash
javac -d . kafka/examples/consumer/MultiThreadHLConsumer.java
```

运行MultiThreadHLConsumer：

```bash
java kafka.examples.consumer.MultiThreadHLConsumer localhost:2181 testgroup kafkatopic 4
```

### consumer属性

* `group.id`：指定consumer group的唯一标识。
* `consumer.id`：唯一标识consumer。默认值为`null`，不指定时会自动生成。
* `zookeeper.connect`：指定ZooKeeper的连接字符串，格式为`<hostname:port/chroot/path>`。`/chroot/path`为全局ZooKeeper命名空间内的数据位置。
* `client.id`：标识发起请求的客户端。默认值为`${group.id}`。
* `zookeeper.session.timeout.ms`：指定一个时间（毫秒）用于consumer等待ZooKeeper声明失效或重新均衡。默认值为`6000`。
* `zookeeper.connection.timeout.ms`：指定客户端建立ZooKeeper连接的最大等待时间（毫秒）。默认值为`6000`。
* `zookeeper.sync.time.ms`：指定ZooKeeper follower同步leader的时间（毫秒）。默认值为`2000`。
* `auto.commit.enable`：该属性为true时，已经被consumer获取的消息offset会被阶段性提交ZooKeeper中。在consumer失效时新consumer将以提交的offset作为起始位置。默认值为`true`。
* `auto.commit.interval.ms`：指定已被消费的offset提交到ZooKeeper的频率（毫秒）。默认值为`60*1000`。
* `auto.offset.reset`：指定offset值，如果ZooKeeper中有初始offset或者offset超出范围。可选值有：`largest`重新设置为最大的offset；`smallest`重新设置为最小的offset；其他任意值抛出异常。默认值为`largest`。
* `consumer.timeout.ms`：在指定的消息间隔后没有可被消费的消息时向consumer抛出一个异常。默认值为`-1`。

---

参考资料：

[《Learing Apache Kafka-Second Edition》]()
