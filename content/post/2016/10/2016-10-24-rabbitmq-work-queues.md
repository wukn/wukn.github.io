---
author: wukn
catalog: true
date: 2016-10-24T01:00:00Z
tags:
- RabbitMQ
title: '[译][RabbitMQ] Work Queues'
url: /2016/10/24/rabbitmq-work-queues/
---

> RabbitMQ是一个message broker，本质上就是接收producer的消息，传递给consumer，其中可以根据给定的需要设置消息路由、缓存、持久化。

<!--more-->

## 简单队列

最简单的队列如下图，producer将消息推送到queue中，queue将消息传递给consumer。可以有多个producer向同一个queue推送消息，也可以有多个consumer从queue接收消息。

![](/img/post/rabbitmq/work-queues/simple-queue.png)

用Java开发producer和consumer需要下载Java client library。

producer代码如下：

```java
// Send.java

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class Send {

  private final static String QUEUE_NAME = "SimpleQueue";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("127.0.0.1");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
    String message = "Hello World!";
    channel.basicPublish("", QUEUE_NAME, null, message.getBytes("UTF-8"));
    System.out.println(" [x] Sent '" + message + "'");

    channel.close();
    connection.close();
  }
}
```

consumer代码如下：

```java
// Recv.java

import com.rabbitmq.client.*;

import java.io.IOException;

public class Recv {

  private final static String QUEUE_NAME = "SimpleQueue";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("127.0.0.1");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    Consumer consumer = new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body)
          throws IOException {
        String message = new String(body, "UTF-8");
        System.out.println(" [x] Received '" + message + "'");
      }
    };
    channel.basicConsume(QUEUE_NAME, true, consumer);
  }
}
```

编译并运行：

```
javac -cp rabbitmq-client.jar Send.java Recv.java

java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Send

java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Recv
```

运行结果如下：

```
[rabbitmq-java-code]$ javac -cp rabbitmq-client.jar Send.java Recv.java
[rabbitmq-java-code]$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Send
 [x] Sent 'Hello World!'
[rabbitmq-java-code]$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Recv
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'Hello World!'
```

## 工作队列（work quque）

任务队列是queue将消息分发给多个consumer进行处理。

![](/img/post/rabbitmq/work-queues/work-queue.png)


```java
// Task.java

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class Task {

  private final static String QUEUE_NAME = "TaskQueue";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("127.0.0.1");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
    String message = argv[0];
    channel.basicPublish("", QUEUE_NAME, null, message.getBytes("UTF-8"));
    System.out.println(" [x] Sent '" + message + "'");

    channel.close();
    connection.close();
  }
}
```

```java
// Worker.java

import com.rabbitmq.client.*;

import java.io.IOException;

public class Worker {

  private final static String QUEUE_NAME = "TaskQueue";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("127.0.0.1");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    Consumer consumer = new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body)
          throws IOException {
        String message = new String(body, "UTF-8");
        System.out.println(" [x] Received '" + message + "'");

        try {
          doWork(message);
        } finally {
          System.out.println(" [x] Done");
        }
      }
    };
    channel.basicConsume(QUEUE_NAME, true, consumer);
  }

  private static void doWork(String task) throws InterruptedException {
    try {
      Thread.sleep(1000);
    } catch (InterruptedException _ignored) {
      Thread.currentThread().interrupt();
    }
  }
}
```

编译代码：

```
javac -cp rabbitmq-client.jar Task.java Worker.java
```

分别打开三个console，其中两个运行Worker：

```
java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Worker
```

另一个console运行Task发送消息：

```
java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Task msg
```

运行结果如下：

```
[rabbitmq-java-code]$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Task msg1
 [x] Sent 'msg1'
[rabbitmq-java-code]$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Task msg2
 [x] Sent 'msg2'
[rabbitmq-java-code]$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Task msg3
 [x] Sent 'msg3'
[rabbitmq-java-code]$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Task msg4
 [x] Sent 'msg4'
[rabbitmq-java-code]$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Task msg5
 [x] Sent 'msg5'
```

```
[rabbitmq-java-code]$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Worker
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'msg1'
 [x] Done
 [x] Received 'msg3'
 [x] Done
 [x] Received 'msg5'
 [x] Done
```

```
[rabbitmq-java-code]$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar Worker
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'msg2'
 [x] Done
 [x] Received 'msg4'
 [x] Done
```

使用任务队列可以实现并行任务处理。默认情况下，RabbitMQ会按消息的顺序依次分发给consumer，也就是说平均每个consumer会接收相同数量的消息。这种消息分发方式称为循环调度（round-robin dispatching）。

## 消息反馈（message acknowledgment）

如果consumer在执行某个消息的任务过程中失败了，会怎么样呢？在当前模式下，RabbitMQ一旦将消息发送给了consumer就从内存中删除该消息，这种情况下，如果杀掉了worker，那么分配给该worker的未处理的和正在处理中的消息就会丢失。但是我们肯定不希望任务丢失，我们希望一个worker挂掉之后，它未处理完成的任务将分配给其他worker处理。

为了确保消息不会丢失，RabbitMQ支持消息反馈。消息反馈就是consumer告诉RabbitMQ某个消息已经被接收并处理完成了，可以从队列中删除了。

如果一个consumer挂掉了（channel或connection被关闭，或者TCP连接丢失），没有发送反馈（ack），那么RabbitMQ就知道消息没有被处理完成，就把它重新放入队列。这时如果有其他consumer在线，那么就把这个消息重新分发。这样就确保在worker挂掉时消息也不会丢失了。

消息处理是没有超时机制的。RabbitMQ会在consumer挂掉时才重新分发消息。因此，即使消息处理时间很长很长都没有关系。

默认消息反馈是开启的。在上面的例子中通过指定`autoAck=true`关闭了反馈。消息反馈的代码如下：

```java
final Consumer consumer = new DefaultConsumer(channel) {
  @Override
  public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
    String message = new String(body, "UTF-8");

    System.out.println(" [x] Received '" + message + "'");
    try {
      doWork(message);
    } finally {
      System.out.println(" [x] Done");
      channel.basicAck(envelope.getDeliveryTag(), false);
    }
  }
};
```

PS：可以使用rabbitmqctl查看未反馈的消息。

```
rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

## 消息持久化

之前的消息反馈可以确保consumer挂掉时消息不会丢失，但是如果RabbitMQ服务器停止或者崩溃了，队列和其中的消息就都丢失了。要确保服务器上的消息不丢失，需要将队列和消息都标识成持久化的。

首先，确保RabbitMQ不会丢失队列：

```java
boolean durable = true;
channel.queueDeclare("TaskQueue", durable, false, false, null);
```

尽管这代码是正确的，但是目前的环境下却不起作用。因为之前已经声明了一个非持久化的队列TaskQueue。RabbitMQ不允许用不同的参数重新定义已经存在的队列。换一个新的队列名称就行了。

```java
boolean durable = true;
channel.queueDeclare("DurableTaskQueue", durable, false, false, null);
```

这样RabbitMQ重启后队列DurableTaskQueue是仍然存在的。接下来需要将消息标识成持久化的，通过设置属性MessageProperties为PERSISTENT_TEXT_PLAIN：

```java
import com.rabbitmq.client.MessageProperties;

channel.basicPublish("", "task_queue",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
```

PS：将消息标识成持久化不能完全保证消息不会丢失。RabbitMQ在接收到消息写入磁盘之间仍有很短的时间间隔，而且RabbitMQ不会每次保存消息时都调用fsync，因此可能消息只被写入了缓存而还没有写入磁盘。如果需要更强的持久化方案，可以使用publisher confirm。

## 公平分配（fair dispatch）

默认的循环调度方案可能会造成负载不平均。例如，有两个worker，偶数号的任务处理很耗时，而奇数号的任务处理很快，那么就会导致一个worker处理很慢，积压很多任务，而另一个worker处理很快，比较空闲。这是因为RabbitMQ在消息进入队列后就立刻分发出去，它不关心consumer中未反馈的消息数量，只是简单地将第n个消息分发给第n个consumer。

为了让每个consumer的负载均匀，通过basicQos方法设置参数prefetchCount为1。这样，consumer中的消息将不超过一条，当consumer处理完并发送反馈后RabbitMQ才会向它分发新的消息。RabbitMQ在接收到新消息后，如果有空闲的consumer就将消息发给它，否则就一直等待有空闲的consumer再分发。

```java
int prefetchCount = 1;
channel.basicQos(prefetchCount);
```

PS：如果所有的任务都很耗时，worker都很忙，那么可能会导致队列被塞满，这时需要增加worker，或者采用其他策略。

最后的完整代码如下：

```java
// Task.java

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.MessageProperties;

public class Task {

  private static final String TASK_QUEUE_NAME = "task_queue";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);

    String message = argv[0];

    channel.basicPublish("", TASK_QUEUE_NAME,
        MessageProperties.PERSISTENT_TEXT_PLAIN,
        message.getBytes("UTF-8"));
    System.out.println(" [x] Sent '" + message + "'");

    channel.close();
    connection.close();
  }

```

```java
// Worker.java

import com.rabbitmq.client.*;

import java.io.IOException;

public class Worker {

  private static final String TASK_QUEUE_NAME = "task_queue";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    final Connection connection = factory.newConnection();
    final Channel channel = connection.createChannel();

    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    int prefetchCount = 1;
    channel.basicQos(prefetchCount);

    final Consumer consumer = new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body, "UTF-8");

        System.out.println(" [x] Received '" + message + "'");
        try {
          doWork(message);
        } finally {
          System.out.println(" [x] Done");
          channel.basicAck(envelope.getDeliveryTag(), false);
        }
      }
    };
    channel.basicConsume(TASK_QUEUE_NAME, false, consumer);
  }

  private static void doWork(String task) {
    try {
      Thread.sleep(1000);
    } catch (InterruptedException _ignored) {
      Thread.currentThread().interrupt();
    }
  }
}
```

---

参考资料：

[RabbitMQ-Queue](http://www.rabbitmq.com/tutorials/tutorial-one-java.html)

[RabbtMQ-Work Queues](http://www.rabbitmq.com/tutorials/tutorial-two-java.html)
