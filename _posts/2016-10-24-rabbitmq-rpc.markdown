---
layout:     post
title:      "[RabbitMQ] Remote Procedure Call"
subtitle:   ""
date:       2016-10-24 03:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - RabbitMQ

---

> 使用RabitMQ构建RPC应用。

## callback queue

RPC的过程为：客户端发送请求消息，服务器相应请求，回复响应消息。客户端为了能接收到相应消息，需要在请求消息中包含回调队列地址。回调队列可以使用默认的队列（在Java中是排他的）。

```java
callbackQueueName = channel.queueDeclare().getQueue();

BasicProperties props = new BasicProperties
                            .Builder()
                            .replyTo(callbackQueueName)
                            .build();

channel.basicPublish("", "rpc_queue", props, message.getBytes());
```

## message properties

最常用的消息属性为：

* deliveryMode：标记消息是否为持久化的。值为MessageProperties.PERSISTENT_TEXT_PLAIN表示是持久的，其他值表示是瞬时的。
* contentType：描述mime-type的编码。如JSON格式该属性为application/json。
* replyTo：通常用来命名回调队列。
* correlationId：关联RPC的请求和响应。

## correlation id

上面的代码为每个RPC请求创建了一个回调队列，这有点不合适，会影响效率和资源占用。可以为每个客户端创建一个回调队列。这又带来一个新的问题，如何知道这个回调队列里的响应消息对应的是哪个请求的？这是就可以使用correlationId属性。我们为每个请求设置唯一的correlationId，当接收到回调队列里的响应时，根据响应的correlationId找到匹配的请求。如果有不识别的correlationId值，那么就丢弃它，说明它不属于该客户端的。这里，直接忽略不识别的消息，而不是抛出错误是因为，很小的可能会出现这样的情景：服务器在发送响应消息后和在向请求队列发送反馈消息前的间隔时间里挂掉了，服务器重启后会再次处理该请求。所以客户端应该合适地处理重复的响应。

## RPC

![](/img/post/rabbitmq/rpc/rpc.png)

1.客户端启动，创建一个匿名的回调队列。

2.客户端发送RPC请求时，在消息中包含两个属性，replyTo回调队列，correlationId唯一标识每个请求。

3.将请求消息发送到请求队列中。

4.RPC服务器接收请求队列中的消息，将处理结果发送到消息的replyTo指定的回调队列中。

5.客户端接受回调队列中的消息，如果消息的correlationId与请求是匹配的，那么将响应消息返回给应用。

完整代码如下：

```java
// RPCServer.java

import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.QueueingConsumer;
import com.rabbitmq.client.AMQP.BasicProperties;

public class RPCServer {

  private static final String RPC_QUEUE_NAME = "rpc_queue";

  private static int fib(int n) {
    if (n ==0) return 0;
    if (n == 1) return 1;
    return fib(n-1) + fib(n-2);
  }

  public static void main(String[] argv) {
    Connection connection = null;
    Channel channel = null;
    try {
      ConnectionFactory factory = new ConnectionFactory();
      factory.setHost("127.0.0.1");

      connection = factory.newConnection();
      channel = connection.createChannel();

      channel.queueDeclare(RPC_QUEUE_NAME, false, false, false, null);

      channel.basicQos(1);

      QueueingConsumer consumer = new QueueingConsumer(channel);
      channel.basicConsume(RPC_QUEUE_NAME, false, consumer);

      System.out.println(" [x] Awaiting RPC requests");

      while (true) {
        String response = null;

        QueueingConsumer.Delivery delivery = consumer.nextDelivery();

        BasicProperties props = delivery.getProperties();
        BasicProperties replyProps = new BasicProperties
                                         .Builder()
                                         .correlationId(props.getCorrelationId())
                                         .build();

        try {
          String message = new String(delivery.getBody(),"UTF-8");
          int n = Integer.parseInt(message);

          System.out.println(" [.] fib(" + message + ")");
          response = "" + fib(n);
        }
        catch (Exception e){
          System.out.println(" [.] " + e.toString());
          response = "";
        }
        finally {
          channel.basicPublish( "", props.getReplyTo(), replyProps, response.getBytes("UTF-8"));

          channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
        }
      }
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
// RPCClient.java

import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.QueueingConsumer;
import com.rabbitmq.client.AMQP.BasicProperties;
import java.util.UUID;

public class RPCClient {

  private Connection connection;
  private Channel channel;
  private String requestQueueName = "rpc_queue";
  private String replyQueueName;
  private QueueingConsumer consumer;

  public RPCClient() throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("127.0.0.1");
    connection = factory.newConnection();
    channel = connection.createChannel();

    replyQueueName = channel.queueDeclare().getQueue();
    consumer = new QueueingConsumer(channel);
    channel.basicConsume(replyQueueName, true, consumer);
  }

  public String call(String message) throws Exception {
    String response = null;
    String corrId = UUID.randomUUID().toString();

    BasicProperties props = new BasicProperties
                                .Builder()
                                .correlationId(corrId)
                                .replyTo(replyQueueName)
                                .build();

    channel.basicPublish("", requestQueueName, props, message.getBytes("UTF-8"));

    while (true) {
      QueueingConsumer.Delivery delivery = consumer.nextDelivery();
      if (delivery.getProperties().getCorrelationId().equals(corrId)) {
        response = new String(delivery.getBody(),"UTF-8");
        break;
      }
    }

    return response;
  }

  public void close() throws Exception {
    connection.close();
  }

  public static void main(String[] argv) {
    RPCClient fibonacciRpc = null;
    String response = null;
    try {
      fibonacciRpc = new RPCClient();

      System.out.println(" [x] Requesting fib(30)");
      response = fibonacciRpc.call("30");
      System.out.println(" [.] Got '" + response + "'");
    }
    catch  (Exception e) {
      e.printStackTrace();
    }
    finally {
      if (fibonacciRpc!= null) {
        try {
          fibonacciRpc.close();
        }
        catch (Exception ignore) {}
      }
    }
  }
}
```

运行结果如下：

```
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar RPCServer
 [x] Awaiting RPC requests
 [.] fib(30)
```

```
[rabbitmq-java-code]$ java -cp .:rabbitmq-client.jar RPCClient
 [x] Requesting fib(30)
 [.] Got '832040'
```

---

参考资料

[RabbitMQ-Remote Procedure Call](https://www.rabbitmq.com/tutorials/tutorial-six-java.html)
