---
title: "[代码整洁之道]错误处理"
author: "wukn"
tags: ["Code"]
catalog: true
date: 2018-06-06T00:00:00+08:00
url: /2018/06/06/clean-code-error-handling/

---

> 错误处理只不过是编程时必须要做的事之一。当错误发生时，程序员就有责任确保代码正常工作。

<!--more-->

许多程序完全由错误处理所占据。所谓占据，并不是说错误处理就是全部。但是几乎无法看明白代码所做的事，因为到处都是凌乱的错误处理代码。错误处理很重要，但如果它搞乱了代码逻辑，就是错误的做法。

### 使用异常而非错误码

在以前，许多语言都不支持异常。这些语言处理和回报错误的手段都非常有限。要么设置一个错误标识，要么返回给调用者检查的错误码。这类手段的问题在于，它们搞乱了调用者代码。调用者必须在调用之后即刻检查错误。不幸的是，这个步骤很容易被遗忘。所以，遇到错误时，最好抛出一个异常。调用代码很整洁，其逻辑不会被错误处理搞乱。

### 先写Try-Catch-Finally语句

异常的妙处之一是，它们在程序中定义了一个范围。执行try-catch-finally语句中的try部分的代码时，是在表明可随时取消执行，并在catch语句中接续。

在某种意义上，try代码块就像是事务。catch代码块将程序维持在一种持续状态，无论try代码块中发生了什么均如此。所以在编写可能抛出异常的代码时，最好先写出try-catch-finally语句。这能帮助我们定义代码的用户应该期待什么，无论try代码块中执行的代码出什么错都一样。

### 使用不可控异常

Java在引入可控异常（checked exception）时，看似是个好点子。每个方法的签名都列出它可能传递给调用者的异常。而且这异常就是方法类型的一部分，如果签名与代码实际所做之事不符，代码在字面上就无法编译。然而，C#、C++、Python和Ruby都不支持可控异常，用它们依然能写出强固的软件。可控异常的代价是违反开放闭合原则。如果在方法中抛出可控异常，而catch语句在三个层级之上，就得在catch语句和抛出异常处之间的每个方法签名中声明该异常。这意味着对软件中较低层级的修改，都将涉及较高层级的签名。

### 给出异常发生的环境说明

抛出的每个异常，都应当提供足够的环境说明，以便判断错误的来源和处所。在Java中，可以从每个异常里得到track trace，但是track trace却无法告诉我们该失败操作的初衷。

应创建信息充分的错误消息，并和异常一起传递出去。在消息中，包括失败的操作和失败类型。如果应用程序有日志系统，传递足够的信息给catch块，并记录下来。

### 依调用者需要定义异常类

当我们在应用程序重定义异常类时，最重要的考虑应该是它们如何被捕获。

看个不太好的例子。

```java
ACMEPort port = new ACMEPort(12);

try {
    port.open();
} catch (DeviceResponseException e) {
	reportPortError(e);
	logger.log("Device response exception", e);
} catch (ATM1212UnlockException e) {
	reportPortError(e);
	logger.log("Unlock exception", e);
} catch (GMXError e) {
	reportPortError(e);
	logger.log("Device response exception");
} finally {
	...
}
```

语句中包含一大堆重复代码。在本例中，可以通过打包调用API、确保返回通用异常类型，从而简化代码。

```java
LocalPort port = new LocalPort(12);

try {
	port.open();
} catch (PortDeviceFailure e) {
	reportError(e);
	logger.log(e.getMessage(), e);
} finally {
	...
}
```

```java
public class LocalPort {
	private ACMEPort innerPort;

	public LocalPort(int portNumber) {
		innerPort = new ACMEPort(portNumber);
	}

	public void open() {
		try {
			innerPort.open();
		} catch (DeviceResponseException e) {
			throw new PortDeviceFailure(e);
		} catch (ATM1212UnlockException e) {
			throw new PortDeviceFailure(e);
		} catch (GMXError e) {
			throw new PortDeviceFailure(e);
		}
}
```

这种将第三方API打包的方式可以降低对它的依赖，未来可以不太痛苦地改用其它代码库。

一般来说，单一异常类通常可行。伴随异常发送出来的信息能够区分不同错误。如果想要捕获某个异常，并且放过其它异常，那就使用不同的异常类。

### 别返回null值

返回null值，会让代码中充斥着if判断，而且只要有一处没检查null值，应用程序就会失控。

如果在代码中返回null值，不如抛出异常，或是返回特例对象（对异常行为进行封装的一个特别的类）。如果调用的第三方API中有返回null值的方法，可以考虑用新方法打包这个方法，在新方法中抛出异常或返回特例对象。

### 别传递null值

将null值传递给方法是一种很糟糕的做法，除非API要求向它传递null值。传入null值会让方法中存在if判断，如果遗漏判断的话就会得到运行时错误。所以，恰当的做法就是禁止传入null值。这样在编码的时候就会时时记着参数列表中的null值意味着出问题了。

### 小结

整洁代码是可读的，但也要强固。可读与强固并不冲突。如果将错误处理隔离看待，独立于主要逻辑之外，就能写出强固而整洁的代码。做到这一步，我们就能单独处理它，也极大地提升了代码的可维护性。

---

参考资料：

《代码整洁之道》——Robert C. Martin
