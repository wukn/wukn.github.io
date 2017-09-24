---
author: wukn
catalog: true
date: 2016-10-24T00:00:00Z
tags:
- RabbitMQ
title: '[译][RabbitMQ] Introduction'
url: /2016/10/24/rabbitmq-introduction/
---

> RabbitMQ是一个messaging broker，一个消息传递的中介。

<!--more-->

## 特性

* 可靠性：RabbitMQ提供了一些特性可以提高可靠性，但会损失一些性能。如消息持久化、消息传递反馈、发布确认、高可用（High Availability）。
* 灵活的路由：消息在到达队列之前经过exchange进行路由。
* 集群：本地网络内的多个RabbitMQ服务器可以组成集群，形成一个逻辑broker。
* 联邦模式：对于需要比集群更松散和更不可靠连接的服务器，RabbitMQ提供了联邦模式。
* 高可用队列：队列可以在集群的多个服务器建立镜像，确保硬件故障时消息也不会丢失。
* 多协议：RabbitMQ支持多种消息协议。
* 管理工具：RabbitMQ提供了管理工具用于监控和控制broker。
* 跟踪：RabbitMQ提供了firehose特性，用于跟踪系统的运行情况，例如查看发布的消息、查看未反馈的消息等。
* 插件系统：RabbitMQ提供了一系列插件来扩展功能，也可以自己编写插件。

## 安装

0.安装Erlang

RabbitMQ使用Erlang开发，因此运行RabbitMQ之前需要安装Erlang环境。

1.下载RabbitMQ

```
wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.0/rabbitmq-server-generic-unix-3.6.0.tar.xz
xz -d rabbitmq-server-generic-unix-3.6.0.tar.xz
tar -xvf rabbitmq-server-generic-unix-3.6.0.tar
cd rabbitmq_server-3.6.0/
```

2.配置RabbitMQ

RabbitMQ默认使用的端口为：

* 4369 (epmd), 25672 (Erlang distribution)
* 5672, 5671 (AMQP 0-9-1 without and with TLS)
* 15672 (启用管理插件时)
* 61613, 61614 (启用STOMP时)
* 1883, 8883 (启用MQTT时)

RabbitMQ提供了三种配置方式：

* 环境变量：定义端口号、文件位置和名称。在shell指定或者在$RABBITMQ_HOME/etc/rabbitmq/rabbitmq-env.conf文件中设置。
* 配置文件：定义服务组件配置，如权限、集群、插件设置等。通常在$RABBITMQ_HOME/etc/rabbitmq/rabbitmq.config文件中。
* 运行时参数：在运行时动态修改集群相关的参数。

3.启动RabbitMQ服务器

```
sbin/rabbitmq-server
```

broker会创建一个默认用户，用户名为guest，密码为guest。未配置的客户端将会使用该用户凭证。默认情况下，该用户凭证时能在本机连接broker时使用。可以创建心用户或者允许guest用户远程访问。

4.管理broker

查看状态：

```
sbin/rabbitmqctl status
```

停止服务：

```
sbin/rabbitmqctl stop
```

5.配置系统限制

在生产环境中使用RabbitMQ可能需要调整系统限制和内核参数来处理并发连接和队列。最主要的参数就是最大打开文件数量，也就是ulimit -n。Linux下默认值是1024，对于messaging broker是远远不够的。建议生产环境的rabbitmq用户允许至少65536个文件描述符。开发环境下4096应该就足够了。

---

参考资料

[www.rabbitmq.com](http://www.rabbitmq.com/install-generic-unix.html)
