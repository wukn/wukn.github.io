---
title: "[微服务]基于Dubbo的分布式服务"
author: "wukn"
tags: ["MicroService", "Dubbo"]
catalog: true
date: 2018-07-21T00:00:00+08:00
url: /2018/07/21/microservice-dubbo-based-distribute-service/

---

> 在互联网场景下应对大流量、高并发，服务化架构改造必不可少。一个拥有较低侵入性的分布式调用跟踪系统来实现服务治理是很重要的，这有助于我们梳理清楚服务或服务之间的依赖关系、调用顺序，以及服务的执行耗时等。

<!--more-->

### 分布式系统的架构演变过程

一般来说，网站由小到大的过程，几乎都要经历单机架构、集群架构、分布式架构。伴随着业务系统架构一同演变的还有各种外围系统和存储系统，如关系型数据库的分库分表改造、从本地缓存过渡到分布式缓存等。

#### 单机系统

对于一个刚上线的项目，往往会将Web服务器、文件服务器和数据库全部部署在同一台服务器上。这对于用户流量较小的网站是完全够用的，即使系统宕机影响范围也不大。但是随着业务开始加速发展，用户逐渐增多，系统瓶颈开始暴露，这是就要考虑对网站架构做出一些调整：

* 独立部署，避免不同的系统之间相互争夺共享资源

* Web服务器集群，实现可伸缩性

* 部署分布式缓存系统，使查询操作尽可能在缓存命中

* 数据库实施读写分离，实现高可用架构

#### 集群架构

在这个阶段主要解决的问题是提升业务系统的并行处理能力，降低单机系统负载，以便支撑更多的用户。

集群技术是将多台独立的服务器通过网络相互连接起来，形成一个有效的整体对外提供服务。互联网领域有一个共同的认知，那就是不要Scale Up，而是Scale Out，也就是说当一台服务器的处理能力接近或已超出其上限时，不要企图更换一台性能更强劲的服务器，而是采用集群技术，通过增加心的服务器来分散并发访问流量，从而实现可伸缩性和高可用架构。

对于无状态的Web节点来说，通过Nginx来实现负载均衡调度似乎是个不错的选择，但生产环境中也要考虑Nginx的高可用性，这可以依靠DNS轮询来实现。伴随着Web集群的改造还有分布式缓存和数据库。对于查询操作应该尽可能在缓存命中，从而降低数据库的负载压力。尽管缓存技术可以解决数据库的大部分查询压力，但是写入操作和无法在缓存命中的数据仍然要频繁地对数据库进行读写操作，因此对数据库实施读写分离改造也迫在眉睫。

业务发展到一定阶段后必然会更加复杂，用户规模也会线性上升。这是可以考虑对网站架构做出以下调整：

* 利用CDN加速系统响应，让静态资源的请求不会落到企业的数据中心

* 业务垂直化，降低耦合，实现分而治之

#### 拆系统之业务垂直化

尽管业务系统可以通过集群技术来提升并行处理能力和实现高可用架构，但是一般到了这个阶段，不仅用户规模暴增，业务需求也变得更加复杂。由于业务逻辑全部耦合在一起，并部署在同一个Web容器中，这必然导致系统中不同业务之间耦合过于紧密，除了扩展和运维困难，还有可能因为某个业务功能不可用而影响系统整体服务不可用。因此这个阶段要解决的问题就是降低业务耦合，提升系统容错性。

所谓拆系统就是指业务垂直化，根据系统业务功能的不同拆分出多个业务模块，由不同的业务团队负责承建，分而治之，独立部署。

拆分粒度越细，耦合越小，容错性越好，每个业务子系统的职责就越清晰。但是如果拆分粒度过细，维护成本将是一个不小的挑战。通常以业务功能维度拆分出多个子系统。

### 服务化与RPC协议

服务化只是一个抽象概念，而RPC协议才是用于实现服务调用的关键。RPC由客户端（服务调用方）和服务端（服务提供方）两部分构成，和在同一个进程空间内执行本地方法相比，RPC的实现细节会相对更复杂。简单来说，服务提供方所提供的方法需要由服务调用方以网络的形式进行远程调用，这个过程也成为RPC请求，服务提供方根据服务调用方提供的参数执行指定的服务方法，执行完成后再将执行结果响应给服务调用方，这样一次RPC调用就完成了。

服务化框架的核心就是RPC。常见的RPC实现方案很多，如Java RMI、WebService、Hessian、Finagle等。不同的RPC实现对序列化和反序列化的处理也不尽相同。将对象序列化成XML/JSON等文本格式具备良好的可读性、扩展性和通用性，但是报文体积大，解析速度慢。在一些注重性能的场景下，采用二进制协议更合适。

### 阿里分布式服务框架Dubbo

Dubbo是阿里贡献给Apache的一个分布式服务框架。

![Dubbo服务调用](/img/post/dubbo/dubbo-service-invoking.svg)

Provider作为服务提供方对外提供服务，当JVM启动时Provider会被自动加载和启动。Provider不需要依赖任何Web容器，因此它可以运行在任何一个普通的Java程序中。当Provider成功启动后，会向注册中心注册指定的服务。服务调用方Consumer在启动后向注册中心订阅目标服务（服务提供者的地址列表），然后在本地根据负载均衡算法从地址列表中选择其中的某一个服务节点进行RPC调用，如果调用失败则自动Failover到其他服务节点上。

在服务调用过程中还有一个不容忽视的问题，就是监控系统，监控系统有助于开发人员分析和定位问题。Dubbo提供了一套完善的监控中心，使我们能清楚地知道服务的状态信息，包括服务调用成功次数、服务调用失败次数、平均响应时间等。这些数据由Provider和Consumer负责统计，实时提交到监控中心。

#### 注册中心

Dubbo官方建议生产环境使用Zookeeper作为注册中心。除此之外还可以使用Redis。

这里我们使用docker启动一个zookeeper容器作为测试，为了方便，启动zookeeper容器的网络模式选择为host。

```bash
docker run -d --name zookeeper --net host -it zookeeper
```

#### Provider服务

首先初始化一个SpringBoot工程，然后在pom中添加dubbo依赖：

```xml
        <dependency>
            <groupId>com.alibaba.boot</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>0.2.0</version>
        </dependency>
```

接下来就是定义服务接口和服务实现：

服务接口：

```java
public interface UserService {
    boolean login(String username, String password);
}
```

服务实现：

```java
public class UserServiceImpl implements UserService {

    @Override
    public boolean login(String username, String password) {
        boolean result = false;
        if ("admin".equals(username)) {
            if ("123456".equals(password)) {
                result = true;
            }
        }
        return result;
    }
}
```

服务接口需要同时包含在服务提供方和服务调用方两端，而服务实现对于服务调用方来说是隐藏的，因此它只需要包含在服务提供方即可。

接下来是为Consumer服务配置Dubbo。这里使用注解的方式。为服务实现添加`@Service`注解：

```java
import com.alibaba.dubbo.config.annotation.Service;

@Service
public class UserServiceImpl implements UserService {
    ...
}
```

除此之外还有一些公共的配置，如zookeeper地址、服务名称等，在配置文件中进行配置：

```yml
dubbo:
  scan:
    basePackages: io.github.wukn.demo.dubbo.service
  application:
    id: dubbo-provider-service
    name: dubbo-provider-service
  registry:
    id: zookeeper
    address: zookeeper://localhost:2181
  protocol:
    id: dubbo
    name: dubbo
    port: 8081
```

`dubbo.scan.basePackages`指定需要扫描的包。`dubbo.application`配置应用名称等。`dubbo.registry`配置注册中心的地址。`dubbo.protocol`配置服务消费者与服务提供者之间通讯的协议。

编写SpringBoot应用引导类：

```java
@SpringBootApplication
public class DubboProviderServiceApplication {

    public static void main(String[] args) {
        new SpringApplicationBuilder(DubboProviderServiceApplication.class).web(WebApplicationType.NONE).run(args);
    }
}
```

启动应用后可以在控制台日志中看到注册的信息，也可以在zookeeper中看到：

```
[zk: localhost:2181(CONNECTED) 15] ls /dubbo/io.github.wukn.demo.dubbo.service.UserService/configuratproviders   
[dubbo%3A%2F%2F192.168.1.9%3A8081%2Fio.github.wukn.demo.dubbo.service.UserService%3Fanyhost%3Dtrue%26application%3Ddubbo-provider-service%26dubbo%3D2.6.2%26generic%3Dfalse%26interface%3Dio.github.wukn.demo.dubbo.service.UserService%26methods%3Dlogin%26pid%3D6815%26side%3Dprovider%26timestamp%3D1532234367915]
```

#### Consumer服务

首先初始化一个SpringBoot工程，然后在pom中添加dubbo依赖：

```xml
        <dependency>
            <groupId>com.alibaba.boot</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>0.2.0</version>
        </dependency>
```

服务消费者只需要定义服务接口即可。实际项目中可以把服务接口的定义放到单独的模块中，让Provider和Consumer都引用它：

服务接口：

```java
public interface UserService {
    boolean login(String username, String password);
}
```

配置dubbo相关参数：

```yml
spring:
  application:
    name: dubbo-consumer-service
dubbo:
  scan:
    basePackages: io.github.wukn.demo.dubbo.service
  application:
    id: dubbo-consumer-service
    name: dubbo-consumer-service
  registry:
    id: zookeeper
    address: zookeeper://localhost:2181
```

在需要用到`UserService`服务的类中注入服务实例，使用`@Reference`注解：

```java
    @Reference
    private UserService userService;
```

简单地测试一下服务调用：

```java
            boolean result = userService.login("admin", "123456");
            System.out.println("is user admin can login: " + result);
            result = userService.login("wukn", "666666");
            System.out.println("is user wukn can login: " + result);
```

运行后可以看到日志中打印出：

```
is user admin can login: true
is user wukn can login: false
```

说明Consumer对Provider的RPC调用成功！

示例的源码在这里[wukn/springboot-dubbo-demo](https://github.com/wukn/springboot-dubbo-demo/tree/0.0.1)。

#### 警惕Dubbo因超时和重试引起的系统雪崩

为了避免系统在大流量场景下产生雪崩现象，一般在对数据库、分布式缓存进行读写操作时会设置超时时间。超时时间如果设置太长，那么应用获取不到结果，又长时间不肯返回，这会对业务产生较大的影响。超时时间设置的太短又适得其反。所以要根据实际的业务场景而定，可以通过日常的压测进行评估。

Dubbo的Consumer在本地根据负载均衡算法从地址列表中选择某一个服务节点进行RPC调用，如果调用失败则自动Failover到其他服务节点上，默认的重试次数为两次，服务调用超时就意味着调用失败，需要进行重试。因此需要注意，如果一些复杂业务本身耗时就很长，但超时时间被不合理地设置为小于服务执行的实际时间，那么在大流量场景下，系统的负载压力将被逐步放大。假设有1000个并发请求同时对服务A进行RPC调用，但都超时失败，由于Failover机制，共将产生3000次并发请求对服务A进行调用。

还有一点，并不是任何类型的服务都适合Failover的。比如写服务，由于要考虑幂等性，调用失败后不应该进行重试，否则将导致数据重复写入。只有读服务开启Failover才有意义。除了Failover，Dubbo还提供了其他容错方案：

* **failover**：服务调用失败时，重试其他服务节点，通常用于读操作

* **failfast**：只发起一次调用，失败立即报错，通常用于非幂等性的写操作

* **failsafe**：失败安全，出现异常时直接忽略，通常用于写入审计日志等操作

* **failback**：失败自动恢复，后台记录失败请求，定时重发，通常用于消息通知操作

* **forking**：并行调用多个服务节点，成功一个即返回，通常用于实时性要求较高的读操作

### 服务治理

准确地说，Dubbo不仅是一个RPC框架，它更是一个服务治理框架。服务治理的三个基本要素是：服务的动态注册与发现；服务的扩容评估；服务的升降级处理。

为什么要引入注册中心？当服务变得越来越多，如果把服务的调用地址配置在服务调用方，那么URL的配置管理将十分麻烦，因此引入注册中心的目的就是实现服务的动态注册和发现，让服务的位置更加透明，并且在客户端实现负载均衡和Failover可以大大降低对硬件负载均衡的依赖。

部署监控中心是为了更好地掌握服务的状态信息，有了这些统计数据后我们就能准确地知道服务的热度。这个数据可以作为扩容的参考指标。

服务的升降级处理也很重要。在一些特殊场景下，如果系统容量在支撑核心业务时都捉襟见肘，那么完全可以对一些次要服务进行降级处理，牺牲部分功能来保证系统的核心业务不受影响。当某些新上线的服务存在bug时也可以采用服务降级的手段来确保系统的整体稳定性。

Dubbo提供了监控中心和管理控制台。包含的服务治理功能有：路由规则、动态配置、服务降级、访问控制、权重调整及负载均衡等。

---

参考资料：

《人人都是架构师——分布式系统架构落地与瓶颈突破》——高翔龙

[dubbo官方文档——注解配置](http://dubbo.apache.org/#!/docs/user/configuration/annotation.md?lang=zh-cn)

[dubbo-spring-boot-starter](https://github.com/apache/incubator-dubbo-spring-boot-project)
