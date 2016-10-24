---
layout:     post
title:      "[RabbitMQ] Publish/Subscribe、Routing、Topic"
subtitle:   ""
date:       2016-10-24 02:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - RabbitMQ

---

> 发布订阅模式下，一个消息会被传递给多个consumer。

## publish/subscribe

### exchange

首先看一下RabbitMQ完整的消息模型。RabbitMQ的核心是producer不直接向producer发送消息，而是只向exchange发送。exchange接收来自producer的消息，再将消息推送到queue中。exchange必须知道如何处理消息，例如，是将消息推到某个特定的queue还是多个queue中，或者直接丢弃？消息的处理规则通过exchange type指定。exchange type有四种：direct，topic，headers和fanout。

![](/img/post/rabbitmq/publish-subscribe/publish-subscribe.png)

fanout exchange就是将所有消息广播给所有它所知道的queue。

```java
channel.exchangeDeclare("logs", "fanout");
```

PS：查看exchange列表的命令为rabbitmqctl list_exchanges。

在之前的队列模式中，我们并没有指定exchange：

```java
channel.basicPublish("", "hello", null, message.getBytes());
```

其中第一个参数就是exchange名称，为空表示使用默认的无名称的exchange：将消息路由到名称为参数routingKey的queue。

现在可以使用excahnge：
```java
channel.basicPublish( "logs", "", null, message.getBytes());
```

### temporary queue

在队列模式中，我们为queue指定了名称。这样consumer可以连接指定的queue，而且producer和consumer可以共享同一个queue。这样创建的队列在服务器重启后仍然存在。

如果每当连接到RabbitMQ时，需要的是新的空队列，在consumer断开连接时自动删除队列，那么可以使用临时队列。临时队列的名称是随机生成的，它是非持久化的、排他的、自动删除的：

```java
String queueName = channel.queueDeclare().getQueue();
```

### binding

exchange和queue之间的关系成为绑定（binding）。查看binding列表的命令是`rabbitmqctl list_bindings`。

建立绑定的方法为：

```java
channel.queueBind(queueName, "logs", "");
```

完整代码如下：

```java
// EmitLog.java

import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;

public class EmitLog {

  private static final String EXCHANGE_NAME = "logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("127.0.0.1");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

    String message = argv[0];

    channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes("UTF-8"));
    System.out.println(" [x] Sent '" + message + "'");

    channel.close();
    connection.close();
  }
}
```

```java
// ReceiveLogs.java

import com.rabbitmq.client.*;

import java.io.IOException;

public class ReceiveLogs {
  private static final String EXCHANGE_NAME = "logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("127.0.0.1");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
    String queueName = channel.queueDeclare().getQueue();
    channel.queueBind(queueName, EXCHANGE_NAME, "");

    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    Consumer consumer = new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope,
                                 AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body, "UTF-8");
        System.out.println(" [x] Received '" + message + "'");
      }
    };
    channel.basicConsume(queueName, true, consumer);
  }
}
```

编译代码并运行：

```
javac -cp .:rabbitmq-client.jar EmitLog.java ReceiveLogs.java

java -cp .:rabbitmq-client.jar EmitLog log1
```
运行结果如下：

```
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar EmitLog log1
 [x] Sent 'log1'
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar EmitLog log2
 [x] Sent 'log2'
```

```
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar ReceiveLogs
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'log1'
 [x] Received 'log2'
```

## routing

在上面的例子中，消息会广播给所有的consumer。有些情况下，我们希望订阅部分消息。如记录日志，只将关键的错误消息写入文件，同时将所有消息输出到console。

### binding key

创建绑定的时侯，可以指定参数routingKey。该参数的作用取决于exchange type。如fanout exchange会忽略该参数。

```java
String routingKey = "black";
channel.queueBind(queueName, EXCHANGE_NAME, routingKey);
```

### direct exchange

direct exchange的路由算法很简单：一个消息的routing key与队列的binding key匹配时，就将该消息推送到这些队列中。可以有多个队列绑定同一个route key。

![](/img/post/rabbitmq/publish-subscribe/direct-exchange.png)

完整代码如下：

```java
// EmitLogDirect.java

import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;

public class EmitLogDirect {

  private static final String EXCHANGE_NAME = "direct_logs";

  public static void main(String[] argv) throws Exception {

    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("127.0.0.1");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "direct");

    String severity = argv[0];
    String message = argv[1];

    channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes("UTF-8"));
    System.out.println(" [x] Sent '" + severity + "':'" + message + "'");

    channel.close();
    connection.close();
  }

  private static String getSeverity(String[] strings){
    if (strings.length < 1)
            return "info";
    return strings[0];
  }

}
```

```java
// ReceiveLogsDirect.java

import com.rabbitmq.client.*;

import java.io.IOException;

public class ReceiveLogsDirect {

  private static final String EXCHANGE_NAME = "direct_logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("127.0.0.1");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "direct");
    String queueName = channel.queueDeclare().getQueue();

    if (argv.length < 1){
      System.err.println("Usage: ReceiveLogsDirect [info] [warning] [error]");
      System.exit(1);
    }

    for(String severity : argv){
      channel.queueBind(queueName, EXCHANGE_NAME, severity);
    }
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    Consumer consumer = new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope,
                                 AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body, "UTF-8");
        System.out.println(" [x] Received '" + envelope.getRoutingKey() + "':'" + message + "'");
      }
    };
    channel.basicConsume(queueName, true, consumer);
  }
}
```

编译并运行，结果如下：

```
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar EmitLogDirect warning log1
 [x] Sent 'log1'
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar EmitLogDirect error log2
 [x] Sent 'log2'
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar EmitLogDirect info log3
 [x] Sent 'log3'
```

```
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar ReceiveLogsDirect warning error
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'warning':'log1'
 [x] Received 'error':'log2'
```

## topic

direct exchange有个限制，就是不能根据多个条件进行路由。

例如，在日志系统中，我们只将来自某工具的错误日志写入文件，同时所有日志输出到console。

### topic exchange

发送到topic exchange的消息的routing key不能是一个单词，而是由.分隔的多个词，如'quick.orange.rabbit'。routing key中的单词数量是任意的，但是总长度不能超过255个字符。binding key也是同样的格式。其中，`*`可以匹配一个单词，#可以匹配0个或多个单词。

例如，我们使用<speed>.<colour>.<species>来描述动物：

![](/img/post/rabbitmq/publish-subscribe/topic-exchange-example.png)

route key为"quick.orange.rabbit"或"lazy.orange.elephant"的消息将会被发送到两个队列中。"quick.orange.fox"只会发送到quque1。"lazy.brown.fox"只会发送到quque2。"lazy.pink.rabbit"只会发送到queue2中一次，尽管它匹配了两个绑定。"quick.brown.fox"不跟任何绑定匹配，该消息会被丢弃。"orange"或"quick.orange.male.rabbit"也会因为不匹配任何绑定而被丢弃。"lazy.orange.male.rabbit"虽然有四个单词，但是与"lazy.#"匹配，会被发送到queue2.

如果一个队列绑定的是`#`，那么它会收到所有消息，就跟fanout exchange一样。
如果绑定中没有使用`*`和`#`，那么这时topic exchange就跟direct exchange一样。

接下来将日志系统改为topic exchange模式，routing key为<facility>.<severity>。

完整代码如下：

```java
// EmitLogTopic.java

import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;

public class EmitLogTopic {

  private static final String EXCHANGE_NAME = "topic_logs";

  public static void main(String[] argv) {
    Connection connection = null;
    Channel channel = null;
    try {
      ConnectionFactory factory = new ConnectionFactory();
      factory.setHost("127.0.0.1");

      connection = factory.newConnection();
      channel = connection.createChannel();

      channel.exchangeDeclare(EXCHANGE_NAME, "topic");

      String routingKey = argv[0];
      String message = argv[1];

      channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes("UTF-8"));
      System.out.println(" [x] Sent '" + routingKey + "':'" + message + "'");
    }
    catch  (Exception e) {
      e.printStackTrace();
    }
    finally {
      if (connection != null) {
        try {
          connection.close();
        }
        catch (Exception ignore) {}
      }
    }
  }
}
```

```java
// ReceiveLogsTopic.java

import com.rabbitmq.client.*;

import java.io.IOException;

public class ReceiveLogsTopic {

  private static final String EXCHANGE_NAME = "topic_logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("127.0.0.1");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "topic");
    String queueName = channel.queueDeclare().getQueue();

    if (argv.length < 1) {
      System.err.println("Usage: ReceiveLogsTopic [binding_key]...");
      System.exit(1);
    }

    for (String bindingKey : argv) {
      channel.queueBind(queueName, EXCHANGE_NAME, bindingKey);
    }

    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    Consumer consumer = new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope,
                                 AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body, "UTF-8");
        System.out.println(" [x] Received '" + envelope.getRoutingKey() + "':'" + message + "'");
      }
    };
    channel.basicConsume(queueName, true, consumer);
  }
}
```

编译后运行结果如下：

```
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar EmitLogTopic "kern.critical" "A critical kernel error"
 [x] Sent 'kern.critical':'A critical kernel error'
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar EmitLogTopic "kern.warning" "A kernel warning"
 [x] Sent 'kern.warning':'A kernel warning'
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar EmitLogTopic "power.warning" "A power warning"
 [x] Sent 'power.warning':'A power warning'
```

```
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar ReceiveLogsTopic "#"
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'kern.critical':'A critical kernel error'
 [x] Received 'kern.warning':'A kernel warning'
 [x] Received 'power.warning':'A power warning'
```

```
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar ReceiveLogsTopic "kern.*"
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'kern.critical':'A critical kernel error'
 [x] Received 'kern.warning':'A kernel warning'
```

```
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar ReceiveLogsTopic "kern.*" "*.critical"
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'kern.critical':'A critical kernel error'
 [x] Received 'kern.warning':'A kernel warning'
```

---

参考资料

[RabbitMQ-Publish/Subscribe](https://www.rabbitmq.com/tutorials/tutorial-three-java.html)

[RabbitMQ-Routing](https://www.rabbitmq.com/tutorials/tutorial-four-java.html)

[RabbitMQ-Topic](https://www.rabbitmq.com/tutorials/tutorial-five-java.html)
