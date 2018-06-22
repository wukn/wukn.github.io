---
title: "[代码整洁之道]类"
author: "wukn"
tags: ["Code"]
catalog: true
date: 2018-06-22T22:00:00+08:00
url: /2018/06/22/clean-code-class/

---

> 除了代码语句及由代码语句构成的函数的表达力，我们还需要关注一下代码组织的更高层面，否则始终得不到整洁的代码。

<!--more-->

### 类的组织

遵循标准的Java约定，类应该从一组变量列表开始。如果有公共静态常量，应该先出现。然后是私有静态变量，以及私有实体变量。很少会有公共变量。

公共函数应跟在变量列表之后。我们喜欢把由某个公共函数调用的私有工具函数紧随在该公共函数后面。这符合了自顶向下原则，让程序读起来就像一篇报纸文章。

我们喜欢保持变量和工具函数的私有性，但并不执着于此。有时，我们也需要用到protected变量或工具函数，好让测试可以访问到。

### 类应该短小

关于类的第一条规则是类应该短小。第二条规则是还要更短小。这条规则跟函数是类似的。那么，多小合适呢？

对于函数，我们通过计算代码行数衡量大小。对于类，我们采用不同的衡量方法，计算**权责**（responsibility）。

类的名称应当描述其权责。如果无法为某个类找到一个精确的名称，这个类大概就太长了。类名越含混，该类越有可能拥有过多权责。例如，如果类名中包括含义模糊的词，如Processor或Manager或Super，这种现象往往说明有不恰当的权责聚集情况存在。

我们也应该能够用大概25个单词描述一个类且不用if、and、or、but等词汇。

#### 单一权责原则

单一权责原则（SRP）认为，类或模块应有且只有一条**加以修改的理由**。该原则既给出了权责的定义，又是关于类的长度的指导方针。鉴别权责（修改的理由）常常帮助我们在代码中认识到并创建出更好的抽象。

系统应该由许多短小的类而不是少量巨大的类组成。每个小类封装一个权责，只有一个修改的原因，并与少数其他类一起协同达成期望的系统行为。

#### 内聚

类应该只有少量实体变量。类中的每个方法都应该操作一个或者多个这种变量。通常而言，方法操作的变量越多，就越黏聚到类上。如果一个类中的每个变量都被每个方法所使用，则该类具有最大的内聚性。

一般创建这种极大化内聚类是既不可取也不可能的。另一方面，我们希望内聚性保持在较高位置。内聚性高，意味着类中的方法和变量互相依赖、互相结合成一个逻辑整体。

保持函数和参数列表短小的策略，有时会导致为一组子集方法所使用的实体变量数量增加。出现这种情况时，往往意味着至少有一个类要从大类中挣扎出来。应当尝试将这些变量和方法分拆到两个或多个类中，让新的类更为内聚。

### 保持内聚性就会得到许多短小的类

仅仅是将较大的函数切割为小函数，就将导致更多的类出现。想想看一个有许多变量的大函数。如果想把该函数中某一小部分拆解成单独的函数，但想要拆出的代码使用了该函数中声明的4个变量。是否必须将这4个变量都作为参数传递到新函数中去呢？

完全没必要！只要将4个变量提升为类的实体变量，完全无需传递任何变量就能拆解代码了。应该很容易将函数拆分为小块。

可惜这也意味着类丧失了内聚性，因为堆积了越来越多只为允许少量函数共享而存在的实体变量。那么，如果有些函数想要共享某些变量，为什么不让它们拥有自己的类呢？当类丧失了内聚性，就拆分它！

所以，将大函数拆分为许多小函数，往往也是将类拆分为多个小类的时机。程序会更加有组织，也会拥有更为透明的结构。

### 为了修改而组织

对于多数系统，修改将一直持续。每处修改都让我们冒着系统其他部分不唔唔如期运行的风险。在整洁的系统中，我们对类加以组织，以降低修改的风险。

下面这个Sql类还没写完，暂时不支持update功能。当需要支持update功能时，我们就得“打开”这个类进行修改。对类的任何修改都有可能破坏类中的其他代码。

```java
public class Sql {
    public Sql(String table, Column[] columns)
    public String create()
    public String insert(Object[] fields)
    public String selectAll()
    public String findByKey(String keyColumn, String keyValue)
    public String select(Column column, String pattern)
    public String select(Criteria criteria)
    public String preparedInsert()
    private String columnList(Column[] columns)
    private String valueList(Object[] fields, final Column[] columns)
    private String selectWithCriteria(String criteria)
    private String placeholderList(Column[] columns)
}
```

当增加一种新语句类型，就要修改Sql类。改动单个语句类型也要进行修改，如让select功能支持子查询。存在两个修改的理由，说明Sql类违反了SRP原则。

可以从一条简单的组织性观点发现对SRP的违反。Sql的方法大纲显示，存在类似selectWithCriteria等只与select语句有关的私有方法。这意味着存在改进空间。

如何解决这些问题呢？将Sql类中的每个接口方法重构到从Sql派生出来的类中。那些私有方法，如valuesList，直接移到需要用它们的地方。公共私有行为划分到独立的两个工具类Where和ColumnList中。

```java
abstract public class Sql {
    public Sql(String table, Column[] columns)
    abstract public String generate()
}

public class CreateSql extends Sql {
    public CreateSql(String table, Column[] columns)
    @override public String generate()
}

public class SelectSql extends Sql {
    public SelectSql(String table, Column[] columns)
    @override public String generate()
}

public class InsertSql extends Sql {
    public InsertSql(String table, Column[] columns, Object[] fields)
    @override public String generate()
    private String valueList(Object[] fields, final Column[] columns)
}

public class SelectWithCriteriaSql extends Sql {
    public SelectWithCriteriaSql(String table, Column[] columns, Criteria criteria)
    @override public String generate()
}

public class SelectWithMatchSql extends Sql {
    public SelectWithMatchSql(String table, Column[] columns, Column column, String pattern)
    @override public String generate()
}

public class FindByKeySql extends Sql {
    public FindByKeySql(String table, Column[] columns, String keyColumn, String keyValue)
    @override public String generate()
}

public class preparedInsertSql extends Sql {
    public preparedInsertSql(String table, Column[] columns)
    @override public String generate()
    private String placeholderList(Column[] columns)
}

public class Where {
    public Where(String criteria)
    public String generate()
}

public class ColumnList {
    public ColumnList(Column[] columns)
    public String generate()
}
```

每个类中的代码都变得极为简单。理解每个类花费的时间缩减到近乎为零。函数对其他函数造成毁坏的风险也变得几近于无。从测试的角度看，验证方案中每一处逻辑都成了极为简单的任务，因为类与类之间互相隔离了。另外，当需要增加update语句时，现存类无需修改。

重构后的代码符合SRP，同时也符合了开放闭合原则（OCP）：类应当对扩展开放，对修改封闭。

#### 隔离修改

需求会改变，所以代码也会改变。具体类包含实现细节，而抽象类则只呈现概念。依赖于具体细节的类，当细节改变时，就会有风险。另一方面，对细节的依赖给对系统的测试带来了挑战。我们借助接口和抽象类来隔离这些细节带来的影响。

通过降低连接度，类遵循了另一条类设计原则，依赖倒置原则（DIP）。本质而言，DIP认为类应当依赖于抽象而不是依赖于具体细节。

---

参考资料：

《代码整洁之道》——Robert C. Martin
