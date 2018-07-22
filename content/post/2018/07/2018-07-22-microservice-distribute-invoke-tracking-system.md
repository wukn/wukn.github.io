---
title: "[微服务]分布式服务调用跟踪系统"
author: "wukn"
tags: ["MicroService", "Dubbo"]
catalog: true
date: 2018-07-22T00:00:00+08:00
url: /2018/07/22/microservice-distribute-invoke-tracking-system/

---

> 在大规模服务调用场景下，服务之间的依赖关系错综复杂，甚至连架构师都无法在短时间内理清服务之间具体的依赖和调用顺序。此外，用户的一次请求可能涉及后端多个服务之间的调用，那些分散在各个机器上的孤立日志对于排查问题显然是不利的。这就是构建分布式调用跟踪系统的意义。

<!--more-->

分布式调用跟踪系统就是一个监控平台，能够以可视化的方式展现跟踪到的每一个请求的完整调用链，并收集调用链上每个服务的执行耗时、整合孤立日志等。业界使用的有淘宝的EagleEye、Twitter的Zipkin、新浪的Watchman、京东的Hydra等。

### Google的Dapper论文

目前众所周知的一些分布式调用跟踪系统大多脱胎于Google的[Dapper](https://bigbully.github.io/Dapper-translation/)论文。这篇论文提到了分布式调用跟踪系统的四个关键设计目标：

* 服务性能低损耗：分布式调用跟踪系统对生产环境中服务的性能损耗应该做到尽可能忽略不计，否则在一些注重性能的场景下会影响系统整体的吞吐量。

* 业务代码低侵入：在开发过程中，业务团队忙于应付大量的新增需求或完善现有功能的缺陷，如果需要在业务中进行大量的埋点上报工作，那就很尴尬了。

* 监控界面可视化：可视化的结果展现对于一个监控系统而言应该是一个并不过分的要求。没有可视化，在后期推广时遇到的阻力可想而知。

* 数据分析准实时：数据的收集、运算和最终结果展现应该做到快速，如果能够达到准实时级别的结果展现，开发人员就能在服务异常的情况下及时做出反应和调整。

#### 如何将调用链串联起来

首先要考虑的是如何将一次请求设计的所有后端服务串联起来，只有这样才能清楚一次请求的完整调用链。论文中，Trace表示对一次请求的完整调用链跟踪；Span为Trace的组成结构，例如服务A和服务B的一次请求/响应过程就是一个Span。一次用户请求可能设计后端多个服务之间的调用，Span就用于体现服务之间具体的依赖关系。

每一次请求都分配一个全局唯一的TraceID，并且整个调用链中所有的Span过程都应该获取到同一个TraceID，以表示这些服务调用过程是发生在同一个Trace上的。

#### 如何埋点和数据收集上报

接下里要考虑的是如何在服务或服务之间的调用过程中进行埋点和数据收集上报。一般来说，在RPC框架中进行埋点上报是最适合不过的，因为这样就可以避免将埋点逻辑侵入业务代码中。埋点位置明确后，要确定收集上报哪些数据，这对后续的计算起决定性作用。一般包括TraceID、SpanID和ParentSpanID、RpcContent中包含的数据信息、服务执行异常时的堆栈信息，以及Trace中各个Span过程的开始时间和结束时间，并且这些数据信息都是需要在Trace的上下文信息中进行传递的。

### 基于Dubbo实现分布式调用跟踪系统方案

Dubbo预留了足够多的接口，可以方便地对其进行二次开发。嵌入在Dubbo中基本上可以做到对业务零侵入，对开发人员尽可能的透明化。

首先Dubbo提供了用于拦截RPC请求的Filter接口，以便实现服务的调用跟踪和数据收集上报等功能扩展。

```java
package io.github.wukn.demo.dubbo.filter;

import com.alibaba.dubbo.rpc.*;

public class TraceFilter implements Filter {
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {

        // 前置拦截
        System.out.println("before invoke");
        RpcContext context = RpcContext.getContext();
        if (null != context) {
            boolean isConsumerSide = context.isConsumerSide(); // 是否是服务调用方
            System.out.println("is consumer side ? " + isConsumerSide);

            boolean isProviderSide = context.isProviderSide(); // 是否是服务提供方
            System.out.println("is provider side? " + isProviderSide);

            String host = context.getLocalHost(); // 获取Host地址
            System.out.println("local host: " + host);

            int port = context.getLocalPort(); // 获取Port
            System.out.println("local port: " + port);

            String serviceName = context.getUrl().getServiceInterface(); // 获取被调用的服务接口名称
            System.out.println("service name: " + serviceName);

            String methodName = context.getMethodName(); // 获取被调用的服务方法名称
            System.out.println("method name: " + methodName);
        }


        // 执行目标服务调用
        Result result = invoker.invoke(invocation);

        // 后置拦截
        System.out.println("after invoke");

        return result;
    }
}
```

编写好Filter后，需要在配置文件中对Filter进行配置：

```yml
dubbo.consumer.filter: traceFilter
```

由于Dubbo的Filter没有纳入Spring的IOC容器中进行管理，因此我们要在/resource目录下创建/META-INF/dubbo/com.alibaba.dubbo.rpc.Filter文件，在文件中通过键值对指定Filter的全限定名：

```
traceFilter=io.github.wukn.demo.dubbo.filter.TraceFilter
```

成功配置好Filter后，当服务调用方想服务提供方发起RPC请求时，Filter会对其进行拦截，开发人员就可以在远程服务方法的执行前后实现自定义的埋点上报逻辑。Dubbo提供的Filter既可以对服务提供方进行拦截，也可以拦截服务调用方。

在Filter中可以使用Dubbo提供的一个临时状态记录器RpcContext类，通过调用RpcContext提供的一系列方法，可以方便地收集到当前Span过程一些信息，如拦截的是服务调用方还是提供方、Host、Port、被调用的服务接口名称，被调用的服务方法名称、用户自定义参数等。

为了得到一个请求的完整调用链，我们要为每个Trace分配一个全局唯一的TraceID，这样一次请求中的每个Span过程都在同一个TraceID下，并且为每个Span过程分配一个SpanID和ParentSpanID。SpanID用于标记每一个Span过程，代表着服务的调用顺序。而Parent SpanID则用于明确Trace服务中的依赖关系。SpanID的值会随每一次服务调用递增，而ParentSpanID的值则来源于上一个Span过程的SpanID。

![通过TraceID串联起来的Trace](/img/post/microservices/trace-graph.svg)

服务调用方如何将这些Trace上下文信息向下传递呢给服务提供方？当Filter对服务调用方进行拦截时，可以将Trace上下文信息Set进由Dubbo提供的RpcInvocation接口中；当Filter对服务提供方进行拦截时，再从中取出之前由服务调用方传递过来的Trace上下文信息即可。

除此之外，还要收集服务执行异常时的堆栈信息，以及各个Span过程的开始时间和结束时间，以便计算服务的执行耗时。当Filter拦截到RPC请求时，记录一个开始时间，当服务完成后记录一个结婚素时间，那么从服务调用方和服务提供方看就是四个的时间戳：

* Client Send Time：CS，客户端发送时间

* Client Receive Time：CR，客户端接收时间

* Server Receive Time：SR，服务端接收时间

* Server Send Time：SS，服务端发送时间

根据这四个时间戳可以计算出每个Span的执行耗时和网络耗时：

* 服务调用耗时 = CR -CS

* 服务处理耗时 = SS -SR

* 网络耗时= 服务调用耗时 - 服务处理耗时

* 前置网络耗时 = SR -CS

* 后置网络耗时 = CR -SS

完整的调用跟踪和数据收集上报流程如下：

```java
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {

        final long BEFORE_TIME = System.currentTimeMillis();
        RpcInvocation rpcInvocation = (RpcInvocation) invocation;

        // 前置拦截
        System.out.println("before invoke");
        boolean isConsumerSide, isProviderSide;
        String host, serviceName, methodName;
        int port;
        RpcContext context = RpcContext.getContext();
        if (null != context) {
            isConsumerSide = context.isConsumerSide(); // 是否是服务调用方
            isProviderSide = context.isProviderSide(); // 是否是服务提供方
            host = context.getLocalHost(); // 获取Host地址
            port = context.getLocalPort(); // 获取Port
            serviceName = context.getUrl().getServiceInterface(); // 获取被调用的服务接口名称
            methodName = context.getMethodName(); // 获取被调用的服务方法名称
        }

        TraceBean traceBean = threadLocal.get(); // TODO: 获取当前线程的Trace上下文信息
        if (isConsumerSide) {
            if (null == traceBean) {
                // TODO： 根调用,需要生成TraceID以创建Trace
            } else {
                final Long TRACE_ID = traceBean.getTraceId();
                // TODO： 递增SpanID，设置ParentSpanID
            }
            // TODO： 将Trace上下文信息设置到Invocation中
        }
        if (isProviderSide) {
            //TODO： 从Invocation中获取服务调用方传来的Trace上下文信息，并设置在ThreadLocal中
        }

        // TODO: 前置数据收集上报

        // 执行目标服务调用
        Result result = invoker.invoke(invocation);

        // 后置拦截
        System.out.println("after invoke");

        // TODO： 后置数据收集上报，以及删除当前线程的Trace上下文信息

        return result;
    }
```


注意，如果跟踪信息直接写入数据库可能会给数据库造成较大的负载压力，建议将消息写入消息队列中，由消费端将消息写入数据库。由于监控数据的时效性较高，长期保存的意义不大，因此定期清理历史数据也是很有必要的。

### 采样率

Dapper论文中提到，关于跟踪系统的四个关键设计目标，其中服务性能低损耗这个设计目标非常重要。跟踪系统的对服务的损耗主要在调用跟踪和数据和收集上。除了优化代码，还可以结合采样率在最大程度上控制损耗。在大流量场景下，通常只需要采样其中很小的一部分请求即可，从而降低跟踪系统带来的服务性能损耗。采样控制最简单的做法是固定的采样率，但这样的做法不够灵活，建议生产环境将采样率配置在配置中心内，以便根据实际的用户流量进行动态调整。

---

参考资料：

《人人都是架构师——分布式系统架构落地与瓶颈突破》——高翔龙
