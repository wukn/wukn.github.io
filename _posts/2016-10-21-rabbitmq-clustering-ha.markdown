---
layout:     post
title:      "[RabbitMQ] Clustering and High Availability"
subtitle:   ""
date:       2016-10-24 04:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - RabbitMQ

---

> 使用RabitMQ构建RPC应用。

## distributed RabbitMQ broker

构建分布式broker有三种方式：clustering、federation、shovel。

* clustering是将多台机器连接在一起形成一个逻辑broker，通过Erlang的消息传递进行通信。集群内所有节点需要有同样的Erlang cookie，也就是说所有的机器运行相同版本的Erlang和RabbitMQ。virtual host、exchange、user、permission会在集群内所有节点之间自动镜像。queue可能只在某一个broker中，也可以配置为在多个节点之间镜像。客户端连接到某个节点后可以访问集群内所有的queue。通常使用clustering来实现高可用和增加吞吐量。
* federation允许一个broker的exchange或queue接收发布到另一个broker的exchange或queue的消息，通过AMQP通信。联邦的exchange之间是单向点对点连接。默认情况下消息只会通过联邦连接传递一次，当然也可以配置成多次来实现复杂的路由。有的消息可能会被联邦连接忽略而不传递，如果一个消息传递到联邦的exchange后不会路由到queue，那么它一开始就不会被传递。联邦queue也是类似的单向点对点连接，消息会根据consumer而被传递任意次数。通常使用federation来连接通过互联网发布/订阅消息和队列的broker。
* shovel概念上跟federation类似，它仅仅是从一个broker的某个queue中消息消息，传递给另一个broker的exchange。shovel提供了比federation更细粒度的控制。

## reliable delivery

网络失效是最常见的故障，而且通常不会立刻被发现。另外，broker和客户端应用可能会遭遇硬件故障。甚至客户端的一些逻辑错误会导致channel或connection错误，需要重新建立channel或connection。这就需要一些措施来确保消息传递的可靠性。

连接失效时，客户端会重新建立新的连接，之前的连接上打开的channel会自动关闭并重新打开。

在连接失效时，正在传输中的消息会丢失。acknowledgement可以解决这个问题。acknowledgement有两种：consumer向server发送它已经接收/处理消息的反馈；server向producer发送同样的反馈。后一种情况在RabbitMQ中成为confirm。

对于操作系统层的TCP连接中断，可以使用heartbeat特性。

为了避免broker中消息丢失，需要考虑broker重启和硬件故障或应用崩溃的情况。

建立持久的exchange和queue并且将消息持久化到磁盘可以保证broker重启后消息不会丢失。

如果要确保broker在硬件故障时仍可用，可以使用clustering。

## clustering

一个RabbitMQ broker是由一个或多个Erlang node组成的一个逻辑组，每个node运行Erlang并共享users、virtual hosts、queues、exchanges、bindings、runtime parameter。也称为cluster。

broker运行所需的所有数据/状态在所有node之间复制，除了queue。queue默认只驻留在某一个node中，但对所有节点是可见且可访问的。要在cluster的node之间复制queue可以使用high availability。

node之间通过域名相互访问，因此需要每个node的hostname可以被其他node解析。（使用操作系统提供的DNS记录、本地host文件或Erlang VM配置。）

cluster有多种配置方式：

* 使用rabbitmqctl手动建立
* 在config文件中声明所有node
* 使用插件rabbitmq-autocluster声明
* 使用插件rabbitmq-clusterer声明

举个例子，构建一个集群，包含三个节点：rabbit1、rabbit2、rabbit3。

RabbitMQ node和命令行工具需要相互通信时，他们必须得有相同的Erlang cookie。Erlang虚拟机在启动RabbitMQ服务器是会自动创建一个随机的cookie文件，通常位于`/var/lib/rabbitmq/.erlang.cookie`或`$HOME/.erlang.cookie`。最简单的方法是，一个node创建了cookie文件后，将它复制到其它node。

### 独立启动每个node

建立cluster是通过将已经存在的node配置成集群，因此首先以正常方式启动所有节点：

```
rabbit1$ rabbitmq-server -detached

rabbit2$ rabbitmq-server -detached

rabbit3$ rabbitmq-server -detached
```

这样，在三个node上创建了独立的broker，使用cluster_status查看结果如下：

```
rabbit1$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit1 ...
[{nodes,[{disc,[rabbit@rabbit1]}]},{running_nodes,[rabbit@rabbit1]}]
...done.

rabbit2$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit2 ...
[{nodes,[{disc,[rabbit@rabbit2]}]},{running_nodes,[rabbit@rabbit2]}]
...done.

rabbit3$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit3 ...
[{nodes,[{disc,[rabbit@rabbit3]}]},{running_nodes,[rabbit@rabbit3]}]
...done.
```

### 创建cluster

为了将三个node连接成一个cluster，将其中两个节点加到第三个节点的cluster中，例如将`rabbit@rabbit2`和`rabbit@rabbit3`加入到rabbit@rabbit1集群中。

将`rabbit@rabbit2`加入`rabbit@rabbit1`集群时，先将`rabbit@rabbit2`上的RabbitMQ应用停掉，再加入`rabbit@rabbit1`集群，最后重启RabbitMQ应用。加入集群后，`rabbit@rabbit2`上的资源和数据将被覆盖。

```
rabbit2$ rabbitmqctl stop_app
Stopping node rabbit@rabbit2 ...done.
rabbit2$ rabbitmqctl join_cluster rabbit@rabbit1
Clustering node rabbit@rabbit2 with [rabbit@rabbit1] ...done.
rabbit2$ rabbitmqctl start_app
Starting node rabbit@rabbit2 ...done.
```

将`rabbit@rabbit3`加入集群：

```
rabbit3$ rabbitmqctl stop_app
Stopping node rabbit@rabbit3 ...done.

rabbit3$ rabbitmqctl join_cluster rabbit@rabbit2
Clustering node rabbit@rabbit3 with rabbit@rabbit2 ...done.

rabbit3$ rabbitmqctl start_app
Starting node rabbit@rabbit3 ...done.
```

使用cluster_status查看结果如下：

```
rabbit1$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit1 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
 {running_nodes,[rabbit@rabbit3,rabbit@rabbit2,rabbit@rabbit1]}]
...done.

rabbit2$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit2 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
 {running_nodes,[rabbit@rabbit3,rabbit@rabbit1,rabbit@rabbit2]}]
...done.

rabbit3$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit3 ...
[{nodes,[{disc,[rabbit@rabbit3,rabbit@rabbit2,rabbit@rabbit1]}]},
 {running_nodes,[rabbit@rabbit2,rabbit@rabbit1,rabbit@rabbit3]}]
...done.
```

使用上面的方法，可以在cluster运行的任意时刻添加node。

### 重启cluster node

加入cluster的node可以随时被停止，也就是说node崩溃不会影响cluster。

依次将`rabbit@rabbit1`和`rabbit@rabbit3`停掉，并查看cluster的状态：

```
rabbit1$ rabbitmqctl stop
Stopping and halting node rabbit@rabbit1 ...done.

rabbit2$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit2 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
 {running_nodes,[rabbit@rabbit3,rabbit@rabbit2]}]
...done.

rabbit3$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit3 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
 {running_nodes,[rabbit@rabbit2,rabbit@rabbit3]}]
...done.

rabbit3$ rabbitmqctl stop
Stopping and halting node rabbit@rabbit3 ...done.

rabbit2$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit2 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
 {running_nodes,[rabbit@rabbit2]}]
...done.
```

再将node重启：

```
rabbit1$ rabbitmq-server -detached
rabbit1$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit1 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
 {running_nodes,[rabbit@rabbit2,rabbit@rabbit1]}]
...done.

rabbit2$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit2 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
 {running_nodes,[rabbit@rabbit1,rabbit@rabbit2]}]
...done.

rabbit3$ rabbitmq-server -detached
rabbit1$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit1 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
 {running_nodes,[rabbit@rabbit2,rabbit@rabbit1,rabbit@rabbit3]}]
...done.

rabbit2$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit2 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
 {running_nodes,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]
...done.

rabbit3$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit3 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
 {running_nodes,[rabbit@rabbit2,rabbit@rabbit1,rabbit@rabbit3]}]
...done.
```

要注意的是：

* 如果cluster所有的node都挂掉了，那么最后一个挂掉的node必须是第一个重新上线的。如果其他node在等待30秒之后，该node仍不能恢复，那么集群就失败了。如果最后一个挂掉的node恢复不了，那么可以使用rabbitmqctl命令的forget_cluster_node将它移除cluster。
* 如果cluster所有的node同时挂掉了（如断电导致的），那么所有node都认为其他node是在它们之后停止的，这是可以使用rabbitmqctl的force_boot来让某个节点可启动。

### 移除cluster node

从cluster中移除一个node：

```
rabbit3$ rabbitmqctl stop_app
Stopping node rabbit@rabbit3 ...done.
rabbit3$ rabbitmqctl reset
Resetting node rabbit@rabbit3 ...done.
rabbit3$ rabbitmqctl start_app
Starting node rabbit@rabbit3 ...done.
```

查看状态如下：

```
rabbit1$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit1 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2]}]},
 {running_nodes,[rabbit@rabbit2,rabbit@rabbit1]}]
...done.

rabbit2$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit2 ...
[{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2]}]},
 {running_nodes,[rabbit@rabbit1,rabbit@rabbit2]}]
...done.

rabbit3$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit3 ...
[{nodes,[{disc,[rabbit@rabbit3]}]},{running_nodes,[rabbit@rabbit3]}]
...done.
```

还可以远程移除node：

```
rabbit1$ rabbitmqctl stop_app
Stopping node rabbit@rabbit1 ...done.

rabbit2$ rabbitmqctl forget_cluster_node rabbit@rabbit1
Removing node rabbit@rabbit1 from cluster ...
...done.
```
这时rabbit1仍然认为它和rabbit2是同一个集群的，直接启动它会报错，需要reset后重新启动：

```
rabbit1$ rabbitmqctl start_app
Starting node rabbit@rabbit1 ...
Error: inconsistent_cluster: Node rabbit@rabbit1 thinks it's clustered with node rabbit@rabbit2, but rabbit@rabbit2 disagrees
rabbit1$ rabbitmqctl reset
Resetting node rabbit@rabbit1 ...done.
rabbit1$ rabbitmqctl start_app
Starting node rabbit@mcnulty ...
...done.
```
现在，三个node是独立的broker了：

```
rabbit1$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit1 ...
[{nodes,[{disc,[rabbit@rabbit1]}]},{running_nodes,[rabbit@rabbit1]}]
...done.

rabbit2$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit2 ...
[{nodes,[{disc,[rabbit@rabbit2]}]},{running_nodes,[rabbit@rabbit2]}]
...done.

rabbit3$ rabbitmqctl cluster_status
Cluster status of node rabbit@rabbit3 ...
[{nodes,[{disc,[rabbit@rabbit3]}]},{running_nodes,[rabbit@rabbit3]}]
...done.
```

值得注意的是，这时`rabbit@rabbit2`仍然保留这集群的状态，而`rabbit@rabbit1`和`rabbit@rabbit3`相当于新建的独立broker。将`rabbit@rabbit2`恢复初始化：

```
rabbit2$ rabbitmqctl stop_app
Stopping node rabbit@rabbit2 ...done.
rabbit2$ rabbitmqctl reset
Resetting node rabbit@rabbit2 ...done.
rabbit2$ rabbitmqctl start_app
Starting node rabbit@rabbit2 ...done.
```

### 在一台机器上构建集群

在一台机器上运行多个RabitMQ node时，每个node需要有独立的节点名称、数据存储位置、日志文件位置、绑定不同的端口。例如：

```
$ RABBITMQ_NODE_PORT=5672 RABBITMQ_NODENAME=rabbit rabbitmq-server -detached
$ RABBITMQ_NODE_PORT=5673 RABBITMQ_NODENAME=hare rabbitmq-server -detached
$ rabbitmqctl -n hare stop_app
$ rabbitmqctl -n hare join_cluster rabbit@`hostname -s`
$ rabbitmqctl -n hare start_app
```

如果除了AMQP端口还有其他端口的话，在启动命令中指定：

```
$ RABBITMQ_NODE_PORT=5672 RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15672}]" RABBITMQ_NODENAME=rabbit rabbitmq-server -detached
$ RABBITMQ_NODE_PORT=5673 RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15673}]" RABBITMQ_NODENAME=hare rabbitmq-server -detached
```

### RAM node

RAM node将一些元数据保存在内存中，减少了写入磁盘的次数，可以提高性能。RAM node可以随转换为disk node。

## high availabile quque

默认情况下，cluster中的queue只存在于某个node中。可以配置queue在多个node之间镜像，建立镜像的queue有一个master和一个或多个slave，当master挂掉时，最早的那个slave将成为新的master。

发布到queue的消息会复制到所有slave中，consumer不管连接的是那个node，它连接的都是master。slave在master受到反馈时也会删除消息。建立queue的镜像提高了可用性，但不是在node之间做负载均衡。

配置queue的镜像策略是通过使用policy。镜像策略在任意时候都是可以修改的。可以创建一个非镜像queue，然后再将它转成镜像queue。非镜像queue和没有slave的镜像queue是不一样的，前者的运行速度更快。配置镜像时需要注意：

* 新的镜像中不包含原master时，RabbitMQ会等到其中一个slave同步完成移除master。例如，queue在[A B]上(其中A是master)现在将它配置到[C D]上，首先会这样[A C D]。一旦同步至新的镜像[C D]后，master A就会被关闭。
* exclusive queue不会建立镜像，因为声明它的connection关闭时它就被删掉了。
新添加到cluster的node根据镜像策略可能会成为slave，这时它不包含queue中已存在的消息，但是会接收新消息的同步，随着时间的推移最终会与master保持完全一致。
* 配置ha-sync-batch-size可设置消息同步的批次大小，可以提高性能。

名称以'ha.'开头的queue镜像到所有node：

```
rabbitmqctl set_policy ha-all "^ha\." '{"ha-mode":"all"}'
```

名称以'two.'开头的queue镜像到任意两个node：

```
rabbitmqctl set_policy ha-two "^two\." '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
```

名称以'nodes.'开头的queue镜像到指定node：

```
rabbitmqctl set_policy ha-nodes "^nodes\." '{"ha-mode":"nodes","ha-params":["rabbit@nodeA", "rabbit@nodeB"]}'
```

---

参考资料

[RabbitMQ documentation](http://www.rabbitmq.com/documentation.html)
