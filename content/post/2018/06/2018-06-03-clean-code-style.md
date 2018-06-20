---
title: "[代码整洁之道]格式"
author: "wukn"
tags: ["Code"]
catalog: true
date: 2018-06-03T10:00:00+08:00
url: /2018/06/03/clean-code-style/

---

> 你应该保持良好的代码格式。应该选用一套管理代码格式的简单规则，然后贯彻这些规则。格式很重要。代码格式关乎沟通，而沟通是专业开发者的头等大事。

<!--more-->

### 垂直格式

源代码文件该有多大？根据统计Junit、FitNesse、testNG、TimeAndMoney、JDepend、Ant和Tomcat代码发现，很多文件少于200行，绝大多数文件不超过500行，但也有一些文件达数千行。这意味着有可能用大多数为200行、最长500行的单个文件构造出出色的系统。短文件通常比长文件易于理解。

#### 向报纸学习

想想看写得好的报纸文章。我们从上到下阅读。在顶部，我们期望有个头条，告诉我们故事主题，好让我们决定是否要读下去。第一段是整个故事的大纲，给出粗线条描述，但隐藏了故事细节。接着读下去，细节渐次增加，直到我们了解所有的日期、名字、引语、说法及其他细节。

源代码也应该这样。名称应当简单且一目了然。名字本身应该足够告诉我们是否在正确的模块中。源文件最顶部应该给出高层次概念和算法。细节应该往下渐次展开，直到找到源文件中最底层的函数和细节。

#### 概念间垂直方向上的区隔

几乎所有的代码都是从上往下读，从左往右读。每行展现一个表达式或一个子句，每组代码展示一条完整的思路。这些思路用空白行区隔开来。

例如，在package声明、import声明和每个函数之间都用空行隔开。每个空白行都是一条线索，标识出新的独立的概念。

#### 垂直方向上的靠近

如果说空白行隔开了概念，那么靠近的代码行则暗示了它们之间的紧密关系/所以，紧密相关的代码应该相互靠近。

#### 垂直距离

你是否曾经在某个类中摸索，从一个函数跳到另一个函数，上下求索，想要弄清楚这些函数如何操作，如何相互相关，最后却被高糊涂了？是否曾经苦苦追索某个变量或函数的继承链条？这令人沮丧，因为你想要理解系统**做什么**，却花时间和精力在找到和记住那些代码碎片**在哪里**。

关系密切的概念应该互相靠近。显然，这条规则并不适用于分布在不同文件中的概念。除非有很好的理由，否则就不要把关系密切的概念放到不同的文件中。

对于那些关系密切、放置于同一源文件中的概念，它们之间的区隔应该成为对相互的易懂度有多重要的衡量标准。

**变量声明**应尽可能靠近其使用位置。因为函数很短，本地变量应该在函数的顶部出现。循环中的控制变量应该总是在循环语句中声明。

**实体变量**应该在类的顶部声明。出现在代码的两个函数中间的实体变量就非常不明显，读者只能在无意中看到这些声明。

**相关函数**。若某个函数调用了另外一个，就应该把它们放到一起，而且调用者应该尽可能放在被调用者上面。这样就能轻易找到被调用的函数，极大地增强了整个模块的可读性。

**概念相关**：概念相关的代码应该放到一起。相关性越强，彼此之间的距离就该越短。

#### 垂直顺序

一般而言，我们想自上而下展示函数调用依赖顺序。也就是说，被调用的函数应该放在执行调用的函数下面。这样就建立了一种自顶向下贯穿代码模块的良好信息流。

像报纸文章一般，我们指望最重要的概念先出来，指望以包括最少细节的方式表述它们。我们指望底层细节最后出来。这样，我们就能扫过源代码文件，自最前面的几个函数获知要旨，而不至于沉溺到细节中。

### 横向格式

一行代码应该有多宽？依旧统计的是上述7个项目。一行字符数在20到60之间的占40%，长于80字符的较少。程序员们显然更喜爱短代码行。

这说明，应该尽力保持代码行短小。但也无需死守80个字符这个上限，达到100或120个字符完全是可以的。再多的话，就有点肆意妄为了。尽量遵循无需拖动滚动条的原则。虽然现在显示器越来越宽，字符可以缩小到屏幕能容纳200个字符的宽度，但是还是建议别超过120个字符。

#### 水平方向上的区隔与靠近

我们使用空格字符将彼此紧密相关的事物连接到一起，也用空格字符吧相关性较弱的事物分隔开来。

在赋值操作符周围加上空格字符，以此达到强调目的。赋值语句有两个确定而重要的要素：左边和右边。空格字符加强了分隔效果。

另一方面，不在函数名和左圆括号之间加空格。这是因为函数与其参数密切相关，如果隔开，就会显得互无关系。函数调用括号中的参数一一隔开，强调逗号，表示参数是互相分离的。

空格字符的另一种用法是强调其前面的运算符。例如`return b*b - 4*a*c`，这样的等式看起来多舒服，然而多数代码格式化工具都会漠视运算符优先级，从头到尾采用同样的空格方式。

#### 水平对齐

有的人喜欢使用这种对齐方式：

```java
public FitNesseExpediter(Socket          s,
                         FitNesseContext context) throws Exception
{
    this.context =            context;
    socket =                  s;
    input =                   s.getInputStream();
    output =                  s.getOutputStream();
    requestParsingTimeLimit = 10000;
}
```

这种对齐方式没什么用。对齐，像是在强调不重要的东西，把读者的目光从真正的意义上拉开。看到上面的声明列表，我们会从上往下阅读变量名，而忽视了它们的类型。赋值语句也一样，我们会从上往下阅读右值，而对赋值运算符视而不见。更烦的是，代码格式化工具会把这类对齐消除掉。相比之下，我更喜欢使用不对齐的声明和赋值。

```java
public FitNesseExpediter(Socket s, FitNesseContext context) throws Exception
{
    this.context = context;
    socket = s;
    input = s.getInputStream();
    output = s.getOutputStream();
    requestParsingTimeLimit = 10000;
}
```

#### 缩进

源文件是一种继承结构，而不是一种大纲结构。其中的信息设计整个文件、文件中每个类、类中的方法、方法中的代码块。也涉及代码块中的代码块，这种继承结构中的每一层级都圈处一个范围，名称可以在其中声明，而声明和执行语句也可以在其中解释。

要让这种范围式继承结构可见，我们依赖源代码行在继承结构中的位置对源代码行做缩进处理。在文件的顶层语句，如大多数类的声明，根本不缩进。类中的方法相对该类缩进一个层级。方法的实现相对方法声明缩进一个层级。代码块的实现相对于其容器代码块缩进一个层级，以此类推。

有缩进结构的文件可以很快地辨别出变量、构造器、存取器、方法等。

有时候，会忍不住想要短小的if语句、while循环或小函数中违反缩进规则。一旦这么做了，多数时候还是会回头家伙丧缩进。这样就避免了出现范围层级坍塌到一行的情况。

```java
public class CommentWidget extends TextWidget
{
    public static final String REGXEP = "^#[^\r\n]*(?:(?:\r\n)|\n|\r)?";

    public CommentWidget(ParentWidget parent, String text) {super()parent, text;}
    public String render() throws Exception {return "";}
}
```

```java
public class CommentWidget extends TextWidget
{
    public static final String REGXEP = "^#[^\r\n]*(?:(?:\r\n)|\n|\r)?";

    public CommentWidget(ParentWidget parent, String text) {
        super()parent, text;
    }
    public String render() throws Exception {
        return "";
    }
}
```

#### 空范围

有时，while或for语句的语句体为空，如下所示。这种结构最好尽量不使用。如果无法避免，就确保空范围体的缩进，用括号包围起来。如果把这个分号放在while语句同一行末尾，那就十分具有欺骗性。除非把那个分号放到另一行再加以缩进，否则就很难看到它。

```java
while (dis.read(buf, 0, readBufferSize) != -1)
;

```

### 团队规则

每个程序员都有自己喜欢的格式规则，但如果在一个团队中工作，就是团队说了算。

一组开发者应当认同一种格式风格，每个成员都应该采用那种风格。我们想要让软件拥有一以贯之的风格。

### Uncle Bob的格式规则

看一下作者给出的他个人所使用规则的一个标准范例：

```java
public class CodeAnalzer implements JavaFileAnalysis {
  private int lineCount;
  private int maxLineWidth;
  private int widestLineNumber;
  private LineWidthHistogram lineWidthHistogram;
  private int totalChars;

  public CodeAnalzer() {
    lineWidthHistogram - new LineWidthHistogram();
  }

  public static List<File> findJavaFiles(File parentDirectory) {
    List<File> files = new ArrayList<File>();
    findJavaFiles(parentDirectory, files);
    return this;
  }

  private static void findJavaFiles(File parentDirectory, List<File> files) {
    for(File file : parentDirectory.listFiles()) {
      if (file.getName().endsWith(".java"))
        files.add(file);
      else if (file.isDirectory())
        findJavaFiles(file, files);
    }
  }

  public void analyzeFile(File javaFile) throws Exception {
    BufferedReader br = new BufferedReader(new FileReader(javaFile));
    String line;
    while (line = br.readLine() != null)
      measureLine(line);
  }

  private void measureLine(String line) {
    lineCount++;
    int lineSize = line.length();
    totalChars += lineSize;
    lineWidthHistogram.addLine(lineSize, lineCount);
    recordWidestLine(lineSize);
  }

  private void recordWidestLine(int lineSize) {
    if (lineSize > maxLineWidth) {
      maxLineWidth = lineSize;
      widestLineNumber = lineCount;
    }
  }

  public int getLineCount() {
    return lineCount;
  }

  public int getMaxLineWidth() {
    return maxLineWidth;
  }

  public int getWidestLineNumber() {
    return widestLineNumber;
  }

  public LineWidthHistogram getLineWidthHistogram)() {
    return lineWidthHistogram;
  }

  public double getMeanLineWidth() {
    return (double) totalChars / lineCount;
  }

  public int getMedianLineWidth() {
    Integer[] sortedWidths = getSortedWidths();
    int cumulativeLineCount = 0;
    for (int width : sortedWidths) {
      cumulativeLineCount += lineCountForWidth(width);
      if (cumulativeLineCount > lineCount / 2)
        return width;
    }
    throw new Error("Cannot get here");
  }

  private int lineCountForWidth(int width) {
    return lineWidthHistogram.getLinesforWidth(width).size();
  }

  private Integer[] getSortedWidths() {
    Set<Integer> widths = lineWidthHistogram.getWidths();
    Integer[] sortedWidths = (widths.toArray(new Integer[0]));
    Arrays.sort(sortedWidths);
    return sortedWidths;
  }
}
```

---

参考资料：

《代码整洁之道》——Robert C. Martin
