---
title: "[代码整洁之道]边界"
author: "wukn"
tags: ["Code"]
catalog: true
date: 2018-06-08T00:00:00+08:00
url: /2018/06/08/clean-code-boundry/

---

> 在软件中，我们通常会使用第三方程序包或使用开源代码，有时候还会使用公司中其他团队打造的组件或子系统。不管哪种情况，我们都得将外来代码干净利落地整合进自己的代码中。

<!--more-->

### 使用第三方代码

以java.util.Map为例，如果应用程序中需要一个放置Sensor类对象的Map，大概会是这样：`Map sensors = new HashMap()`。当代码中需要访问其中的元素时，就会这样：`Sensor s = (Sensor)sensors.get(sensorId)`。这行代码会反复出现，它承担了取对象并转换类型的职责。而且这行代码并未说明自己的用途。通过使用泛型，这段代码可读性可以大大提高。

```java
Map<Sensor> sensors = new HashMap<Sensor>();
...
Sensor s = sensors.get(sensorId);
```

不过，Map<>提供了超出所需的功能的问题还是没有解决。更整洁的方式大致如下：

```java
public class Sensors {
    private Map sensors = new HashMap();

    public Sensor getById(String id) {
        return (Sensor) sensors.get(id);
    }
}
```

边界上的接口是隐藏的。如果Map的接口发生修改（Java 5加入泛型支持时的确发生了改动），那么只需要修改该类就行了。同时对泛型的使用也不再是大问题，因为转换和类型管理都在这个类内部处理的。

### 浏览和学习边界

第三方代码帮助我们在更少时间内发布更丰富的功能。在使用第三方程序包时，该从何入手呢？我们没有测试第三方代码的职责，但为要使用的第三方代码编写测试，可能最符合我们的利益。

设想第三方代码库的使用方法并不清楚。我们可能会花上一两天或更多时间去阅读文档，决定如何使用。然后开始编写使用第三方代码的代码，看是否如我们愿的工作。陷入长时间的调试、找我们或他们代码中的bug，这是常有的事。

学习第三方代码很难。整个第三方死啊嘛也很难。两件事加一起更难。那该怎么做呢？不要在生产代码中试验新东西，而是编写测试来浏览和理解第三方赛马。这叫做**学习性测试**。在学习性测试中，我们如在应用中那样调用代码。通过核对试验来检测自己对那个API的理解程度。测试聚焦于我们想从API得到的东西。

### 学习log4j

比如，我们想使用log4j来代替自定义的日志代码。我们下载了log4j，打开文档介绍页。无需看太久，就编写了第一个测试用例，希望它能向控制台输出hello字样。

```java
@Test
public void testLogCreate() {
    Logger logger = Logger.getLogger("MyLogger");
    logger.info("hello");
}
```

运行，logger发生了一个错误，告诉我们需要用Appender。再读一点文档，发现有个ConsoleAppender。于是我们创建了一个。

```java
@Test
public void testLogCreate() {
    Logger logger = Logger.getLogger("MyLogger");
    ConsoleAppender appender = new ConsoleAppender();
    logger.addAppender(appender);
    logger.info("hello");
}
```

这回，我们发现Appender没有输出流。Google一下之后，写出以下代码。

```java
@Test
public void testLogCreate() {
    Logger logger = Logger.getLogger("MyLogger");
    logger.removeAllAppender();
    logger.addAppender(new ConsoleAppender(
        new PatternLayout("%p %t %m%n"),
        ConsoleAppender.SYSTEM_OUT));
    logger.info("hello");
}
```

这回行了。当我们一处ConsoleAppender.SYSTEM_OUT参数时，仍然能输出hello字样。但是如果取走PatternLayout就会出现没有输出流的错误信息。

再搜索、阅读、测试，最终我们得到如下代码清单。我们极大地发掘了log4j的工作方式，也将得到的知识融入了一系列简单的单元测试中。

```java
public class LogTest {
    private Logger logger;

    @Before
    public void initialize() {
        logger = logger.getLogger("logger");
        logger.removeAllAppender();
        Logger.getRootLogger().removeAllAppender();
    }

    @Test
    public void basicLogger() {
        BasicConfigurator.configure();
        logger.info("basicLogger");
    }

    @Test
    public void addAppenderWithStream() {
        logger.addAppender(new ConsoleAppender(
            new PatternLayout("%p %t %m%n"),
            ConsoleAppender.SYSTEM_OUT));
        logger.info("addAppenderWithStream");
    }

    @Test
    public void addAppenderWithoutStream() {
        logger.addAppender(new ConsoleAppender(
            new PatternLayout("%p %t %m%n")));
        logger.info("addAppenderWithoutStream");
    }
}
```

### 学习性测试的好处不只是免费

学习性测试毫无成本。无论如何我们都要学习要使用的API，而编写测试是获得这些知识的容易而不会影响其他工作的途径。学习性测试是一种精确试验，帮助我们增进对API的理解。

当第三方包发布了新版本，通过运行学习性测试，就可以知道程序的行为有没有改变。学习性测试确保第三方程序包按照我们想要的方式工作。如果第三方程序包的修改与测试不兼容，我们能立马发现。

### 使用尚不存在的代码

还有一种边界，那种将已知和未知分隔开的边界。有时候我们需要的接口是第三方代码还没有提供的。于是我们定义了自己使用的接口，也就是我们希望得到的接口。这样的好处是它在我们控制之下。一旦第三方的API被定义出来，我们就编写一个Adapter来跨接。这个方案为测试提供了极大的方便。在拿到第三方API之前，我们可以使用Mock对象来测试我们的代码。（适配器模式）

### 整洁的边界

边界上的代码需要清晰的分割和定义了期望的测试。应该避免我们的代码过多地了解第三方代码中的特定信息。依靠我们能控制的东西，好过依靠我们控制不了的东西，免得以后受它控制。

我们通过代码中少数几处引用第三方边界接口的位置来管理第三方边界。像对待Map那样包装它们，或者使用适配器模式将我们的接口转换为第三方提供的接口。采用这两种方式，代码都能更好地与我们沟通。

---

参考资料：

《代码整洁之道》——Robert C. Martin
