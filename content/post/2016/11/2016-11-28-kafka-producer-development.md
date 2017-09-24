---
author: wukn
catalog: true
date: 2016-11-28T03:00:00Z
tags:
- Kafka
title: '[译][Kafka]开发Producer'
url: /2016/11/28/kafka-producer-development/
---

> 《Learing Apache Kafka-Second Edition》

<!--more-->

procedure就是产生消息并将消息发布至broker的应用。

![](/img/post/kafka/producer-development/kafka-producer.png)

producer连接至任意的活动节点并请求获取某个topic的partition的leader元数据。这样producer可以直接将信息发给该partition的lead broker。

出于效率考虑，producer可以分批发布消息，但是只能在异步模式下。异步模式下，producer可以配置`queue.time`或`batch.size`这两个参数其中一个来指定在一定数量或一定时间后批量发布消息。消息会在producer这一端积累，然后在一次请求中批量发布至broker。因此异步模式也带来了消息丢失的风险，当producer崩溃时，在内存中的积累的尚未发布的消息就丢失了。

对于异步模式的producer，回调函数可以用来注册捕捉错误的处理器。

### Java producer API

* Producer
Kafka提供了类`kafka.javaapi.producer.Producer`（`class Producer<K, V>`）用于向一个或多个topic创建消息，还可以制定消息的partition。K和V分别指定partiton key和消息的值的类型。

![](/img/post/kafka/producer-development/java-producer-api.png)

* KeyedMessage
类`kafka.producer.KeyedMessage`的构造函数参数为topic名称、partition key和消息值：

```
class KeyedMessage[K,V](val topic: String, val key: K, val message: V)
```

这里，K和V仍然分别是指定partiton key和消息的值的类型，topic始终是String类型的。

* ProducerConfig
类`kafka.producer.ProducerConfig`封装了与broker建立连接所需要的参数，如borker list、partition类、消息序列化类、partiton key。

producer的API封装了同步模式下producer的实现，异步模式下producer基于`producer.type`。例如，异步模式的`kafka.producer.Producer`负责消息序列化和发送之前的数据缓存。在内部，`kafka.producer.async.ProducerSendThread`的实例从队列中读出该批次的消息，`kafka.producer.EventHandler`序列化并发送数据。配置`event.handler`这个参数还可以自定义处理器。

### 一个简单的Java producer

接下来，我们写一个类`SimpleProducer`来创建指定的topic对应的消息，并使用默认的partition。

1.引入以下类：

```java
import kafka.javaapi.producer.Producer;
import kafka.producer.KeyedMessage;
import kafka.producer.ProducerConfig;
```

2.定义属性：

```java
Properties props = new Properties();
props.put("metadata.broker.list", "localhost:9092, localhost:9093, localhost:9094");
props.put("serializer.class", "kafka.serializer.StringEncoder");
props.put("request.required.acks", "1");
ProducerConfig config = new ProducerConfig(props);
Producer<String, String> producer = new Producer<String, String>(config);
```

看一下代码中提到的属性：
* `metadata.broker.list`：该属性指定producer要连接的broker（格式为`[<node:port>, <node:port>]`）。Kafka producer会自动为topic选择lead broker，并且在发布消息时连接到正确的broker。
* `serializer.class`：该属性指定准备发送消息时对消息进行序列化的类。在本例中使用的是Kafka提供的字符串编码器。默认情况下key和消息的序列化类是一样的。也可以通过扩展`kafka.serializer.Encoder`来实现自定义的序列化类。设置参数`key.serializer.class`就可以使用自定义编码器。
* `request.required.acks`：该属性指示broker在收到消息后向producer发送回执。1表示只要lead副本接收到消息就发送回执。

3.构造消息并发送：

```java
String runtime	= new Date().toString();
String msg = "Message Publishing Time - " + runtime;
KeyedMessage<String, String> data = new KeyedMessage<String, String>(topic, msg);
producer.send(data);
```

完整代码如下：

```java
package kafka.examples.producer;

import java.util.Date;
import java.util.Properties;
import kafka.javaapi.producer.Producer;
import kafka.producer.KeyedMessage;
import kafka.producer.ProducerConfig;
public class SimpleProducer {
    private static Producer<String, String> producer;
    public SimpleProducer() {
        Properties props = new Properties();

        // Set the broker list for requesting metadata to find the lead broker
        props.put("metadata.broker.list",
                "192.168.146.132:9092, 192.168.146.132:9093, 192.168.146.132:9094");

        //This specifies the serializer class for keys
        props.put("serializer.class", "kafka.serializer.StringEncoder");

        // 1 means the producer receives an acknowledgment once the lead replica
        // has received the data. This option provides better durability as the
        // client waits until the server acknowledges the request as successful.
        props.put("request.required.acks", "1");

        ProducerConfig config = new ProducerConfig(props);
        producer = new Producer<String, String>(config);
    }
    public static void main(String[] args) {
        int argsCount = args.length;
        if (argsCount == 0 || argsCount == 1)
            throw new IllegalArgumentException(
                    "Please provide topic name and Message count as arguments");
        // Topic name and the message count to be published is passed from the
        // command line
        String topic = (String) args[0];
        String count = (String) args[1];
        int messageCount = Integer.parseInt(count);
        System.out.println("Topic Name - " + topic);
        System.out.println("Message Count - " + messageCount);
        SimpleProducer simpleProducer = new SimpleProducer();
        simpleProducer.publishMessage(topic, messageCount);
    }
    private void publishMessage(String topic, int messageCount) {
        for (int mCount = 0; mCount < messageCount; mCount++) {
            String runtime = new Date().toString();
            String msg = "Message Publishing Time - " + runtime;
            System.out.println(msg);
            // Creates a KeyedMessage instance
            KeyedMessage<String, String> data =
                    new KeyedMessage<String, String>(topic, msg);

            // Publish the message
            producer.send(data);
        }
        // Close producer connection with broker.
        producer.close();
    }
}
```

在运行上面的代码之前，确保已经创建了名为`kafkatopic`的topic：

```bash
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 3 --topic kafkatopic
```

添加环境变量`KAFKA_LIB`指向Kafka的lib文件夹路径，并将lib文件夹下的jar包添加到`classpath`。

编译代码：

```bash
javac -d . kafka/examples/producer/SimpleProducer.java
```

运行程序，`SimpleProducer`接收两个参数，topic名称和消息数量：

```bash
java kafka.examples.producer.SimpleProducer kafkatopic 10
```

之后可以运行consumer接收消息了：

```bash
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic kafkatopic
```

### 自定义partition的Java producer

上面的例子是一个非常简单的针对多broker集群的producer，没有明确指定消息的partition。接下来我们写一个带自定义消息partition的。例子的场景是，捕捉并发布从各个IP访问网站的日志消息。日志消息包含：网站被访问时的timestamp、网站的名称、访问网站的IP地址。

1.引用以下类

```java
import java.util.Date;
import java.util.Properties;
import java.util.Random;
import kafka.javaapi.producer.Producer;
import kafka.producer.KeyedMessage;
import kafka.producer.ProducerConfig;
```

2.定义属性

```java
Properties props = new Properties();
props.put("metadata.broker.list", "localhost:9092, localhost:9093, localhost:9094");
props.put("serializer.class", "kafka.serializer.StringEncoder");
props.put("partitioner.class", "kafka.examples.producer.SimplePartitioner");
props.put("request.required.acks", "1");
ProducerConfig config = new ProducerConfig(props);
Producer<Integer, String> producer = new Producer<Integer, String>(config);
```
属性`partitioner.class`指定用于决定消息发送的topic内partition的类。如果为null，则使用key的哈希值。

3.实现分区类

编写一个自定义分区类`SimplePartitioner`，它是抽象类`Partitioner`的实现。

```java
package kafka.examples.producer;
import kafka.producer.Partitioner;
public class SimplePartitioner implements Partitioner {
    public SimplePartitioner (VerifiableProperties props) {

    }
    /*
        * The method takes the key, which in this case is the IP address,
        * It finds the last octet and does a modulo operation on the number
        * of partitions defined within Kafka for the topic.
        *
        * @see kafka.producer.Partitioner#partition(java.lang.Object, int)
        */
    public int partition(Object key, int a_numPartitions) {
        int partition = 0;
        String partitionKey = (String) key;
        int offset = partitionKey.lastIndexOf('.');
        if (offset > 0) {
            partition = Integer.parseInt(partitionKey.substring(offset + 1))
                    % a_numPartitions;
        }
        return partition;
    }
}
```

4.构造消息并发送

完整代码如下：

```java
package kafka.examples.producer;

import java.util.Date;
import java.util.Properties;
import java.util.Random;
import kafka.javaapi.producer.Producer;
import kafka.producer.KeyedMessage;
import kafka.producer.ProducerConfig;
public class CustomPartitionProducer {
    private static Producer<String, String> producer;
    public CustomPartitionProducer() {
        Properties props = new Properties();
        // Set the broker list for requesting metadata to find the lead broker
        props.put("metadata.broker.list",
                "192.168.146.132:9092, 192.168.146.132:9093, 192.168.146.132:9094");
        // This specifies the serializer class for keys
        props.put("serializer.class", "kafka.serializer.StringEncoder");

        // Defines the class to be used for determining the partition
        // in the topic where the message needs to be sent.
        props.put("partitioner.class", "kafka.examples.ch4.SimplePartitioner");

        // 1 means the producer receives an acknowledgment once the lead replica
        // has received the data. This option provides better durability as the  
        // client waits until the server acknowledges the request as successful.
                props.put("request.required.acks", "1");

        ProducerConfig config = new ProducerConfig(props);
        producer = new Producer<String, String>(config);
    }
    public static void main(String[] args) {
        int argsCount = args.length;
        if (argsCount == 0 || argsCount == 1)
            throw new IllegalArgumentException(
                    "Please provide topic name and Message count as arguments");
        // Topic name and the message count to be published is passed from the
        // command line
        String topic = (String) args[0];
        String count = (String) args[1];
        int messageCount = Integer.parseInt(count);
        System.out.println("Topic Name - " + topic);
        System.out.println("Message Count - " + messageCount);
        CustomPartitionProducer simpleProducer = new CustomPartitionProducer();
        simpleProducer.publishMessage(topic, messageCount);
    }
    private void publishMessage(String topic, int messageCount) {
        Random random = new Random();
        for (int mCount = 0; mCount < messageCount; mCount++) {

            String clientIP = "192.168.14." + random.nextInt(255);
            String accessTime = new Date().toString();
            String message = accessTime + ",kafka.apache.org," + clientIP;
            System.out.println(message);

            // Creates a KeyedMessage instance
            KeyedMessage<String, String> data =
                    new KeyedMessage<String, String>(topic, clientIP, message);

            // Publish the message
            producer.send(data);
        }
        // Close producer connection with broker.
        producer.close();
    }
}
```

在运行上面的代码之前，确保已经创建了名为`website-hits`的topic：

```bash
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 5 --topic website-hits
```

编译代码：

```bash
javac -d . kafka/examples/producer/SimplePartitioner.java
javac -d . kafka/examples/producer/CustomPartitionProducer.java
```

运行程序：

```bash
java kafka.examples.producer.CustomPartitionProducer website-hits 100
```

运行consumer接收消息：

```bash
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic kafkatopic
```

### producer属性

* `metadata.broker.list`：producer使用该属性获取元数据（topic、partition、、replica），格式为`host1:port1,host2:port2`。
* `serializer.class`：指定消息的序列化类。默认值为`kafka.serializer.DefaultEncoder`，。
* `producer.type`：指定消息发送是同步模式还是异步模式。可选值为`async`和`sync`。默认值为`sync`。
* `request.required.acks`：指定producer请求完成时broker是否向producer发送回执。默认值为0。0表示producer不等待broker的回执，这样可以降低延迟，但可靠性降低。1表示在lead副本接收到数据后producer将立即收到回执，这提高了可靠性，因为客户端会等待服务器端处理请求完成的回执。-1表示在所有同步的副本都收到数据后producer将收到回执，这提供了最佳的可靠性。
* `key.serializer.class`：指定对key的序列化类。默认值为`${serializer.class}`。
* `partitioner.class`：指定在topic中对消息进行分区的类。默认值为`kafka.producer.DefaultPartitioner`，是基于key的哈希值。
* `compression.codec`：指定producer压缩数据的格式，可选的值有`none`、`gzip`、`snappy`。默认值为`none`。
* `batch.num.messages`：指定异步模式时批次发送消息的数量。默认值为200。producer会等到消息数量达到该值或者达到`queue.buffer.max.ms`后才会发送消息。

---

参考资料：

[《Learing Apache Kafka-Second Edition》]()
