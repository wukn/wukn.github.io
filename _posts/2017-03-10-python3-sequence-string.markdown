---
layout:     post
title:      "[Python3] 序列（字符串）"
subtitle:   ""
date:       2017-03-10 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Python3

---

> Python3 序列——字符串。

字符串有几种方法来表示：用单引号('...')或双引号("...")括起来。

常规字符串中可以使用`\`转义字符。原始字符串为在字符串开始的引号之前加上`r`，可以取消`\`转义。

```python
>>> print('C:\some\name')  # here \n means newline!
C:\some
ame
>>> print(r'C:\some\name')  # note the r before the quote
C:\some\name
```

字符串可以跨多行。一种方法是使用三引号："""..."""或者'''...'''。行尾换行符会被自动包含到字符串中，但是可以在行尾加上`\`(续行符)来避免这个行为。

字符串可以用+操作符联接，也可以用*操作符重复多次：

```python
>>> 3 * 'un' + 'ium'
'unununium'
```

相邻的两个或多个字符串字面量（用引号引起来的）会自动连接。

```python
>>> 'Py' 'thon'
'Python'
```

这种方式只能用于两个字符串的连接，变量或者表达式是不行的。

```python
>>> prefix = 'Py'
>>> prefix 'thon'  # can't concatenate a variable and a string literal
  ...
SyntaxError: invalid syntax
>>> ('un' * 3) 'ium'
  ...
SyntaxError: invalid syntax
```

除了序列所支持的那些操作，字符串还支持以下字符串操作：

* `str.capitalize()`：返回字符串的一个副本，该副本的首字母大写而其他字母小写
* `str.casefold()`：返回字符串的一个副本，该副本的所有字母小写
* `str.center(width[, fillchar])`：居中str（返回长度为width的字符串：str居中、以fillchar填充两侧）。fillchar默认为空格。width小于len(str)时返回原字符串
* `str.count(sub[, start[, end]])`：返回sub子串在str[start:end]中出现的次数（非重叠）
* `str.encode([encoding[, errors]])`：返回编码格式encoding的str
* `str.endswith(suffix[, start[, end]])`：如果str[start:end]以suffix子串结尾，返回True；否则，返回False。suffix可以是一个元组
* `str.expandtabs([tabsize])`：返回字符串的一个副本，其中所有的tab字符用空格替换并补齐，使得每一个tab字符后的子串位于第tabsize*n列（n=0,1,2,...）
* `str.find(sub[, start[, end]])`：返回sub子串在str[start:end]中的最小索引，不存在则返回-1
* `str.format(*args, **kwargs)`：字符串格式化。将字符串中使用大括号{}定义的替换域（Replacement Field）替换为相应索引的位置参数（Positional Argument）的或者相应名称的关键字参数Keyword Argument）。在替换域中，支持访问参数的属性或成员

```python
>>> "The result of 1 + 2 is {}, 1 * 2 is {}".format(1+2, 1*2)
'The result of 1 + 2 is 3, 1 * 2 is 2'
>>> "The result of 1 + 2 is {0}".format(1+2)
'The result of 1 + 2 is 3'
>>> "The result of 1 + 2 is {result1}, 1 * 2 is {result1}".format(result1=1+2, result2=1*2)
'The result of 1 + 2 is 3, 1 * 2 is 2'
```

* `str.format_map(mapping)`：跟`str.format(**mapping)`相似
* `str.index(sub[, start[, end]])`：返回sub子串在str[start:end]中的最小索引，不存在则抛出ValueError异常
* `str.isalnum()`：如果str不为空且其中仅含有字母或数字，返回True；否则，返回False
* `str.isalpha()`：如果str不为空且其中仅含有字母，返回True；否则，返回False
* `str.isdigit()`：如果str不为空且其中仅含有数字（Digit Character），返回True；否则，返回False
* `str.isidentifier()`：如果根据语言定义str是一个有效的identifier，返回True
* `str.islower()`：如果str至少包含一个字符，且所有字符全部为小写，返回True；否则，返回False
* `str.isnumeric()`：如果str不为空且所有字符都是数字字符（Numeric character），返回True；否则，返回False。数字字符（Numeric character）包含数字（Digit Character）和Unicode数字值属性（Unicode numeric value property），如U+2155, VULGAR FRACTION ONE FIFTH。
* `str.isprintable()`：如果str为空或者所有字符都是可打印的，返回True；否则，返回False
* `str.isspace()`：如果str不为空且其中仅含有空白符，返回True；否则，返回False
* `str.istitle()`：如果str不为空且是标题化的，返回True；否则，返回False
* `str.isupper()`：如果str不为空且所有字符都是大写字母，返回True；否则，返回False
* `str.join(iterable)`：串联迭代对象iterable生成的字符串序列，并以str分隔
* `str.ljust(width[, fillchar])`：类似center()，但左对齐str，以fillchar填充右侧。fillchar默认为空格。如果width小于len(str)，则返回原字符串
* `str.lower()`：将str中的字母全部小写
* `str.lstrip([chars])`：返回字符串的一个副本，该副本中删除开始处、位于chars中的字符（默认为空白符）
* `str.maketrans(x[, y[, z]])`：该静态方法返回一个用于`str.translate()`的翻译表（translation table）
* `str.partition(sep)`：如果str中存在sep，以第一个sep的下标为分界，返回元组 (sep之前的子串, sep, sep之后的子串)；否则，返回元组 (str, '', '')
* `str.replace(old, new[, count])`：将str中的old子串替换为new子串，只替换前count个（省略count则全部替换）
* `str.rfind(sub[, start[, end]])`：逆序版find()：返回最大下标，找不到则返回-1
* `str.rindex(sub[, start[, end]])`：逆序版index()：返回最大下标，找不到则抛出ValueError异常
* `str.rjust(width[, fillchar])`：类似center()，但右对齐str，以fillchar填充左侧
* `str.rpartition(sep)`：逆序版partition()：存在则以最后一个sep的下标为分界，返回元组（sep之前的子串, sep, sep之后的子串)；否则返回 ('', '', str)
* `str.rsplit([sep[, maxsplit]])`：从右向左分割str，以sep为分割符（省略时，sep默认为空白符），只执行maxsplit次分割（省略则全部分割）
* `str.rstrip([chars])`：删除str结尾处、位于chars中的字符（默认为空白符）
* `str.split([sep[, maxsplit]])`：逆序版rsplit()：从左向右分割str
* `str.splitlines([keepends])`：以换行符（universal newlines）为分割符，分割str；如果指定keepends且为True，则分割后保留换行符
* `str.startswith(prefix[, start[, end]])`：逆序版endswith()：判断str[start:end]是否以prefix子串开始
* `str.strip([chars])`：综合lstrip()和rstrip()的功能：同时删除str开始和结尾处的空白符（或指定字符）
* `str.swapcase()`：将str中的所有字母的大小写翻转：大写变小写，小写变大写
* `str.title()`：标题化str：使得str中非字母字符之后的字母（或首字母）大写，字母字符之后的字母小写
* `str.translate(table)`：返回字符串的一个副本，该副本中根据table中的转换关系对字符进行转换
* `str.upper()`：返回字符串的一个副本，该副本中的所有小写字母变为大写
* `str.zfill(width)`：返回字符串的一个副本，该副本的长度为width，左侧补'0'

```python
>>> '42'.zfill(5)
'00042'
>>> '-42'.zfill(5)
'-0042'
```

### 相关模块

Python标准库中跟字符串相关的[模块](https://docs.python.org/3.5/library/text.html)有：

* string：通用字符串操作
* re：正则表达式操作
* difflib：比较序列之间的差异
* textwrap：文本包装和填充
* unicodedata：Unicode字符库
* stringprep：提供用于互联网协议的Unicode字符串
* readline：GNU的readline接口
* rlcompleter：GNU的readline的补齐函数

---

参考资料：

[Python3 Documentation](https://docs.python.org/3.5/library/stdtypes.html#text-sequence-type-str)
