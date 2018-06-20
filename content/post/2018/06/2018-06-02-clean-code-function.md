---
title: "[代码整洁之道]函数"
author: "wukn"
tags: ["Code"]
catalog: true
date: 2018-06-02T00:00:00+08:00
url: /2018/06/02/clean-code-function/

---

> 函数是所有程序中的第一组代码。一起看看如何写好函数。

<!--more-->

### 短小

函数的第一规则是要短小。第二条规则是**还要更短小**。作者的意见是20行封顶最佳。函数越小，每个函数就能一目了然。每个函数都只说一件事。而且，每个函数都依序把读者带到下一个函数。这就是函数应该达到的短小程度。

if语句、else语句、while语句等，其中的代码块应该只有一行。该行应该就是一个函数调用语句。这样不但能保持函数短小，而且，因为块内调用的函数拥有较具说明性的名称，从而增加了文档上的价值。

这也意味着函数不应该大到足以容纳嵌套结构。所以，函数的缩进层级不该多于一层或两层。这样的函数易于阅读和理解。

### 只做一件事

**函数应该做一件事。做好这件事。只做这一件事。**问题在于很难知道那件该做的事是什么。

```java
public static String renderPageWithSetupsAndTeardowns(
    PageData pageData, boolean isSuite) throws Exception {
    if (isTestPage(pageData))
        includeSetupAndTeardownPages(pageData, isSuite);
    return pageData.getHtml();
}
```

上述代码只做了一件事。但是也容易看成是三件事：判断是否为测试页面；如果是，添加设置和销毁步骤；渲染成HTML。注意这三个步骤均在该函数名下的同一抽象层上。可以用简洁的TO语句来描述它：要RenderPageWithSetupsAndTeardowns，检查页面是否为测试页，如果是测试页，就添加设置和销毁步骤，无论是否测试页，都渲染成HTML。

如果函数只是做了该函数名下同一抽象层上的步骤，则函数还是只做了一件事。编写函数毕竟是为了把大一些的概念（也就是函数的名称）拆分为另一抽象层上的一系列步骤。

如果一个函数被切分为多个区段，那就明显它做了太多的事。只做一件事的函数无法被合理地切分为多个区段。

### 每个函数一个抽象层级

要确保函数只做一件事，函数中的语句都要在同一抽象层级上。函数中混杂不同抽象层级，往往让人迷惑。读者可能无法判断某个表达式是基础概念还是细节。更恶劣的是，就像破损的窗户，一旦细节与基础概念混杂，更多的细节就会在函数中纠结起来。

我们想让代码拥有自顶向下的阅读顺序。我们想要让每个函数后面都跟着位于下一抽象层级的函数，这样一来，在查看函数列表时，就能循抽象层级向下阅读了。这叫做向下规则。

### switch语句

写出短小的switch语句很难。即便是只有两种条件的switch语句也要比单个代码块或函数大得多。写出只做一件事的switch语句也很难，switch天生要做N件事。不幸的是我们总无法避开switch语句，不过还是能够确保每个switch都埋藏在较低的抽象层级，而且永远不重复。当然，我们利用多态来实现这一点。

看这段代码，它呈现了可能依赖于员工类型的仅仅一种操作。

```java
public Money calculatePay(Employee e) throws InvalidEmployeeType {
    switch (e.type) {
        case COMMISSIONED:
            return calculateCommissionedPay(e);
        case HOURLY:
            return calculateHourlyPay(e);
        case SALARIED:
            return calculateSalariedPay(e);
        default:
            throw new InvalidEmployeeType(e.type);
    }
}
```

这个函数有几个问题。首先，它太长，出现新的员工类型的时候还会变得更长。其次，它明显做了不止一件事。第三，它违反了单一职责原则，因为有好几个修改它的理由。第四，它违反了开放闭合原则，因为每当添加新类型时，就必须修改它。最麻烦的是在很多地方都可能需要用到类似的结构，如`isPayday(Employee e, Date date)`或`deliverPay(Employee e, Money pay)`。

这个问题的解决方案是将switch语句埋到抽象工厂底下，不让任何人看到。该工厂使用switch为Employee的派生类创建适当的实体，而不同的函数则由Employee接口多态地接受调用。

```java
public abstract class Employee {
    public abstract boolean isPayday();
    public abstract Money calculatePay();
    public abstract void deliverPay(Money pay);
}
```
```java
public interface EmployeeFactory {
    public Employee makeEmployee(EmployeeRecord r) throws InvalidEmployeeType;
}
```
```java
public class EmployeeFactoryImpl implements EmployeeFactory {
    public Employee makeEmployee(EmployeeRecord r) throws InvalidEmployeeType {
        switch (r.type) {
            case COMMISSIONED:
            return new CommissionenEmployee(r);
        case HOURLY:
            return new HourlyEmployee(r);
        case SALARIED:
            return new SalariedEmployee(r);
        default:
            throw new InvalidEmployeeType(r.type);
        }
    }
}
```

### 使用描述性名称

给函数名和方法取个具有描述性的名何曾，能较好地描述要做的事情。记住沃德原则：“如果每个例程都让你感到深合己意，那就是整洁代码”。函数越短小，功能越集中，就越便于取个好名字。

别害怕长名称。长而具有描述性的名称要比短而令人费解的名称好，也比描述性的长注释好

命名方式要保持一致。使用与模块名一脉相承的短语、名词和动词给函数命名。

### 函数参数

最理想的参数数量是0，其次是1，再次是2，应尽量避免3。有足够特殊的理由才能用三个以上参数（这一条好苛刻啊，回忆了一下以往工作，太多三个以上参数的函数了）。参数与函数名处在不同的抽象层级，它要求读者了解目前并不特别重要的细节。

从测试的角度看，要编写能确保参数的各种组合运行正常的测试用例是很难的事。参数越多，问题就越麻烦。

输出参数比输入参数还要难以理解。读函数时，我们习惯于认为信息通过参数输入函数，通过返回值从函数中输出。我们不太期望信息通过参数输出。

#### 一元函数

向函数传入单个参数有两种普遍的理由。可能是问关于那个参数的问题，就像`boolean fileExists("MyFile")`。也可能是操作该参数，将其转换为其他什么东西，再输出。如`InputStream fileOpen("MyFile")`。

还有一种不普遍但极有用的单参数函数形式，那就是事件。它只有输入参数而无输出参数。程序将函数看作是一个事件，使用该参数修改系统状态。应该让读者很清楚地了解它是个事件。谨慎地选用名称和上下文语境。

#### 标识参数

标识参数丑陋不堪。想函数传入布尔值简直就是骇人听闻的做法。这本身就在大声宣布函数做了不止一件事。

#### 二元函数

有两个参数的函数要比一元函数难懂。当然，也有两个参数正好的时候，如`Point p = new Point(0,0);`。这个例子中两个参数其实是单个值的有序组成部分。

像`assertEquals(expected, actual)`这样的函数，多少次我们呢会搞错actual和expected的位置呢？这两个参数没有自然的顺序，是一种需要学习的约定罢了。

二元函数还不算恶劣。但也应该尽量利用一些机制将其转化为一元函数。例如将其中一个参数写成类的成员，从而无需传递它。

#### 参数对象

如果函数看来需要两个、三个或三个以上参数，就说明其中一些参数应该封装成类了。例如：

```java
Circle makeCircle(double x, double y, double radius);
```

```java
Circle makeClrcle(Point center, double radius);
```

当一组参数被同时传递，往往就是该有自己名称的某个概念的一部分。

#### 动词与关键字

给函数取个好名字，能较好地解释函数的意图，以及参数的顺序和意图。对于一元函数，函数和参数应当形成一种非常良好的动词/名词对形式。例如`write(name)`，更好的大概是`writeField(name)`，它告诉我们，name是一个field。

将参数的名称编码进函数名，这被称作函数名称的关键字形式。例如，将assertEqual改成assertExpectedEqualsActual就会好些，能大大减轻记忆参数顺序的负担。

### 无副作用

副作用是谎言。函数承诺只做一件事，但是还是会做其他被藏起来的事。有时它会对自己类中的变量做出未能预期的改动。有时会把变量弄成系统全局变量。这些操作都是具有破坏性的，会导致古怪的时序性耦合及顺序依赖。

### 分隔指令与询问

函数要么做什么事，要么回答什么事，二者不可得兼。两样都干常会导致混乱。例如：

```java
public boolean set(String attribute, String value);
```

该函数设置某个指定属性，如果成功就返回true，如果不存在那个属性就返回false。这就导致了这样的语句`if (set("username", "wukn")) ...`。这是在询问username属性值是否之前已设置为wukn吗？还是问username属性值是否成功设置为wukn呢？从这行调用中难判断其义。因为set是动词还是形容词并不清楚。作者本意set是动词，但在if语句的上下文中感觉它像个形容词。要解决这个问题，可以将set函数重命名为setAndCheckIfExists，但这对提高if语句的可读性帮助不大。真正的解决方案是把指令与询问分隔开来，防止混淆发生：
```java
if (attributeExists("username")) {
    setAttribute("username", "wukn");
}
```

### 使用异常替代返回错误码

从指令式函数返回错误码轻微违反了指令与询问分隔的原则。它鼓励了在if语句判断中把指令当作表达式使用。如`if (deletePage(page) == E_OK)`。这不会引起动词/形容词混淆，但却导致更深层次的嵌套结构。当返回错误码时，就是在要求调用者立刻处理错误。

```java
if (deletePage(page) == E_OK) {
    if (registry.deleteReference(page.name) == E_OK) {
        if (configKeys.deleteKey(page.name.makeKey()) == E_OK) {
            logger.log("page deleted");
        } else {
            logger.log("configKey not deleted");
        }
    } else {
        logger.log("deleteReference from registry failed");
    }
} else {
    logger.log("delete failed");
    return E_ERROR;
}
```

如果使用异常代替返回错误码，错误处理代码就能从主路径代码中分离出来，得到简化：

```java
try {
    deletePage(page);
    registry.deleteReference(page.name);
    configKeys.deleteKey(page.name.makeKey());
} catch (Exception e) {
    logger.log(e.getMessage());
}
```

#### 抽离try/catch代码块

try/catch代码块丑陋不堪。它们搞乱了代码结构，把错误处理与正常流程混为一谈。最好把try和catch代码块的主体部分抽离出来，另外形成函数。

```java
public void delete(Page page) {
    try {
        deletePageAndAllReferences(page);
    } catch (Exception e) {
        logError(e);
    }
}

private void deletePageAndAllReferences(Page page) throws Exception {
    deletePage(page);
    registry.deleteReference(page.name);
    configKeys.deleteKey(page.name.makeKey());
}

private void logError(Exception e) {
    logger.log(e.getMessage());
}错误`
```

上面的代码中，delete函数只与错误处理有关，可以很容易理解然后就忽略掉。deletePageAndAllReferences函数只与删除page有关，错误处理可以忽略掉。通过这样的区分，代码更易于理解和修改。

#### 错误处理就是一件事

函数应该只做一件事。错误处理就是一件事。因此，处理错误的函数不该做其他事。这意味着，如果关键字try在某个函数中存在，它就应该是这个函数的第一个单词，而且在catch/finally代码块后面也不该有其他内容。

#### 依赖磁铁

返回错误码通常暗示某处有个类或是枚举，它定义了所有错误码。这样的类就是一块依赖磁铁（dependency magnet），其他许多类都得导入和使用它。当其进行修改时，所有其他的类都要重新编译和部署。这会导致开发者不愿增加新的错误代码，就不断复用旧的错误码。

使用异常代替错误码，新异常就可以从异常类派生出来，无需重新编译或重新部署。（这也是开放闭合原则的一个范例）

### 别重复自己（DRY）

重复会导致代码臃肿，且当发生变化时需要修改多处。整体代码的可读性因为重复的消除而得到提升。

许多原则和实践规则都是为了控制与消除重复而创建。如关系数据库范式是为了消灭数据重复而服务。面向对象变成将代码集中到基类从而避免冗余。面向切面变成也是消除重复的一种策略。

### 结构化编程

有些程序员遵循结构化编程规则。Dijkstra认为，每个函数、函数中的每个代码块都应该只有一个入口、一个出口。遵循这个规则意味着在每个函数中只该有一个return语句，循环中不能有break或continue，而且永远不能有任何goto语句。

我们赞成结构化编程的目标和规范，但是对于小函数这些规则助益不大。只有在大函数中，这些规则才会有明显的好处。所以只要保持函数短小。偶尔出现的return、break和continue语句没有坏处，甚至还比单入单出原则更具有表达力。另一方面，goto只在大函数中才有道理，所以应该尽量避免使用。

### 如何写出这样的函数

写代码和写论文或文章很像，先想写什么就些什么，然后再打磨它，斟酌推敲，直至到达心目中的样子。

写函数时，一开始都冗长而复杂。有太多缩进和嵌套循环。有过长的参数列表。名称是随意取的，也会有重复代码。但最好还是配上一套单元测试，覆盖每一行丑陋的代码。

然后打磨这些代码，分解函数、修改名称、消除重复。缩短和重新安置方法。有时还拆散类。同时保持测试通过。

最后，遵循上述的规则，组装好这些函数。

### 小结

每个系统都是使用某种领域特定语言搭建，而这种语言是程序员设计来描述那个系统的。函数是语言的动词，类是名词。

大师级程序员吧系统当作故事来讲，而不是当作程序来写。他们使用选定编程语言提供的工具构建一种更为丰富且更具表达力的语言，用来讲那个故事。那种领域特定语言的一个部分，就是描述在系统中发生的各种行为的函数层级。在一种精妙的递归操作中，这些行为使用它们定义的与领域紧密相关的语言讲述自己那个小故事。

如果遵循上述的这些规则，函数就会短小，有个好名字，而且被很好地归置。不过永远别忘记，真正的目标在于讲述系统的故事，而你编写的函数必须干净利落地拼装到一起，形成一种精确而清晰的语言，帮助你讲故事。

---

参考资料：

《代码整洁之道》——Robert C. Martin
