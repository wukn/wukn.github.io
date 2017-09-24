---
author: wukn
catalog: true
date: 2016-10-21T01:00:00Z
tags:
- Architecture
title: "[译][软件架构模式]-事件驱动架构Event-Driven Architecture"
url: /2016/10/21/event-driven-architecture/
---

> 事件驱动架构是个流行的分布式异步架构模式，用于构建高伸缩性的应用。它有很高的适应性，既适用于小型应用，也适合大型复杂应用。事件驱动架构是由高度解耦的、目标单一、可异步接收和处理事件的组件组成。

<!--more-->

该架构主要有两种拓扑结构：mediator和broker。当需要将多个步骤编排到一个事件中时使用mediator，当不使用mediator将多个事件串联起来时使用broker。这两种拓扑结构的架构特征和实现方式是不一样的，你需要知道哪一种是合适你的场景的。

## Meditor拓扑结构

mediator拓扑结构用于这样的场景，事件有多个步骤，步骤之间需要一定层次的编排。例如，一个股票交易事件，首先需要验证交易的有效性，检验交易是否符合规则，将交易指派给经纪人，计算佣金，最终完成交易。这些步骤需要经过一定层次的安排，确定步骤的顺序，哪些步骤是串行执行，哪些步骤可以并行执行。

mediator拓扑结构的架构包括四种类型的组件：事件队列（event queues）、事件中介（event mediator）、事件通道（event channels）、事件处理器（event processors）。事件流从客户端发送事件到时间队列开始，事件队列将事件传送给事件中介，事件中介接收初始事件，发送一些额外的异步事件给事件通道来执行处理的每个步骤。事件处理器监听事件通道，接收来自事件中介的事件，执行业务逻辑来处理事件。

![](/img/post/software_architecture_pattern/event_driven_architecture/mediator_topology.png)
图1 事件驱动架构的mediator拓扑结构(Event-driven architecture mediator topology)

在事件驱动架构中有十几个甚至几百个事件队列都是常见的。该模式并没有限定消息队列组件的实现方式，可以是消息队列（message queue）、web服务端点（web service endpoint）、或者几种方式的结合。

该模式中有两种事件类型，初始事件（initial event）和处理事件（processing event）。初始事件是中介接收到的原始事件，而处理事件是中介生成的、事件处理组件接收的事件。

事件中介组件用于组织原始事件包含的若干步骤。对于初始事件的每一步，中介将发出一个处理事件到某个事件通道中，由事件处理器接收并处理。要注意的是，中介实际上不对初始事件进行逻辑处理，但它知道初始事件需要分成几个步骤来处理。

事件通道是被中介用来异步向时间处理器传递原始事件某个步骤对应的处理事件。事件通道既可以是消息队列（message queues）也可以是消息主题（message topics），尽管在mediator拓扑结构中广泛使用消息主题，以便处理事件可以被多个事件处理器处理。

事件处理组件包含处理事件所需的业务逻辑。每个处理器是自包含的，独立的，高度解耦的，执行单一的任务。虽然事件处理组件的粒度既可以很小也可以很大，但是每个事件处理组件应该可以处理一个单独的业务任务而不需要依赖其他事件处理组件来一起完成。

事件中介有很多种实现方式。作为架构师，你需要理解每个实现的细节，确保选择与业务需求相合适的方式。

最简单和常用的是开源的集成组件如Sping Integration、Apache Camel、Mule ESB。它们的事件流通常是Java语言或DSL（domain-specific language）。更复杂的有BPEL（business process execution language）加上BPEL引擎，如开源的Apache ODE。BPEL是类XML语言，用来描述处理初始事件所需要的数据和步骤。更大型的应用中复杂的事件编排时可以使用BPM（business process manager）如jBPM。

举个例子，假设在在保险公司进行了投保，现在决定搬家。初始事件可能名为relocation event，该事件中包含的一系列步骤如图2所示。对于初始事件的每一步，事件中介都创建一个处理事件，将处理事件发送到事件通道，等待相应的事件处理器进行处理。其中部分处理事件是可以同时执行的。

![](/img/post/software_architecture_pattern/event_driven_architecture/mediator_topology_example.png)
图2 mediator拓扑结构案例

## broker拓扑结构

broker与mediator的区别在于没有中心的事件中介，与之相反，消息流在链式的轻量级消息代理（message broker）（如RabbitMQ，ActiveMQ，HornetQ）中的事件处理组件中穿过。这种拓扑结构在事件流相对比较简单且不想用（或不需要）集中式的事件编排时很有用。

在broker拓扑结构中主要有两个组件：代理（broker）组件和事件处理器（event processor）组件。代理组件可以是集中式的，也可以是联邦式的，它包含事件流中所有的事件通道。代理组件中的事件通道可以是消息队列（message queues）或消息主题（message topics）或两者的结合。

![](/img/post/software_architecture_pattern/event_driven_architecture/broker_topology.png)
图3 事件驱动架构的broker拓扑结构(Event-driven architecture broker topology)

从broker拓扑结构图中可以看出，没有中央的事件中介组件来控制和编排初始事件，而是每个事件处理组件处理一个事件并发布一个新事件表明它进行处理过的动作。例如，一个平衡股票组合投资的事件处理器收到初始事件stock split，针对该初始事件，时间处理器可能做一些投资组合再平衡，然后向代理发布一个新事件rebalance protfolio，由其它处理器进行处理。注意有时可能出现事件处理器发布事件，但没有其他处理器接收事件。这在更新应用或提供一些未来的功能和扩展时是很常见的。

用与mediator相同的例子来说名broker结构。由于broker拓扑结构中没有了中央的事件中介来接收初始事件，customer-process组件直接接收事件，改变用户的住址，然后发送change address事件表示已改变用户住址。本例中有两个事件处理器可以接收处理change address，quote process和claims process。quote process组件基于改变的地址重新计算保险的费率，并发布recalc quote事件表明它已经处理完成；claims process组件更新保险索赔并发布update claim事件。新的事件被其它组件接收并处理，最终事件链穿过整个系统直到没有新事件发布了。（PS：不懂国外的情况，这个例子真心没看懂...）

![](/img/post/software_architecture_pattern/event_driven_architecture/broker_topology_example.png)
图4 broker拓扑结构案例

从图4中可以看出，broker拓扑结构就是处理业务的事件链。可以将它看成接力赛一样，一个选手带着接力棒跑一段距离后交给下一位选手，知道所有选手都完成比赛。在接力赛中，一个选手交出了接力棒就意味着他完成了比赛。

## 注意事项

事件驱动架构模式实现起来相对比较复杂，主要是因为它的异步分布式的特性。实现该模式时，需要处理大量分布式架构的问题，如远程处理的可用性，缺乏响应，失败时的broker重连问题等。

选择这种模式的一个需要考虑的是对单一的业务逻辑缺乏原子事务性。因为事件处理组件是高度解耦的和分布式的，通过他们共同完成一个事务单元是十分困难的。因此，选择这种模式设计应用时，必须不断考虑哪些事件可以或不可以独立运行，相应地规划事件处理器的粒度。

或许该模式最困难的是事件处理组件之间的契约的创建、维护和管理。每个事件都有一个对应的契约（传递给事件处理器的数据值和数据格式）。在一开始就选定数据格式（XML、JSON、Java Object等）和建立契约版本策略是非常重要的。

## 模式分析
下表展示了分层架构模式的通用架构特性的评级和分析。

整体灵活性

评级：高

分析：整体灵活性是对环境变化的快速响应能力。由于事件吹组件是单目标的，而且相互之间完全解耦，通常变化被隔离在组件内或几个事件处理器之间，可以不影响其他组件快速地被修改。

易于部署

评级：高

分析：由于事件组件之间相互解耦，该模式部署起来相对比较容易。broker拓扑结构比mediator托布结构更容易部署，因为事件中介从某种程度上来说和事件处理器是耦合的，修改事件处理组件时可能也需要修改事件中介，需要同时部署事件处理组件和事件中介。

可测试性

评级：低

分析：虽然独立的单元测试并不是十分困难，但是需要特殊的测试客户端或测试工具来生成事件。异步的特性也导致测试比较复杂。

性能

评级：高

分析：正如所有的消息架构，也许实现的架构性能不是很好，但由于异步的特性，该架构的性能通常比较高。换句话说，执行解耦的、并行的异步操作的能力超过消息队列出入的成本。

可伸缩性

评级：高

分析：该模式的可扩展性来自于事件处理器的高度独立和解耦。每个事件处理器可以被独立地扩展，可扩展的粒度很小。

开发容易度

评级：低

分析：异步性、契约的创建、事件处理器未响应或代理失败时需要更多的错误处理，使得开发过程比较复杂。

---

参考资料

[Software Architecture Patterns](http://www.oreilly.com/programming/free/software-architecture-patterns.csp)
