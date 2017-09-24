---
author: wukn
catalog: true
date: 2017-03-11T00:00:00Z
tags:
- Python3
title: '[译][Python3] 序列（列表、元组、Range）'
url: /2017/03/11/python3-sequence-list-tuple-range/
---

> Python3 序列——列表、元组、Range。

<!--more-->

### 列表（List）

列表是可变序列，常用于存储相同性质对象的集合。实际上存储的对象可以是任何类型的对象，包括Python标准类型（数值、字符串、列表、元组和字典）以及用户自定义类型（类）。

#### 列表构建方式

列表的创建方式有以下几种：
* 空列表：`[]`
* 非空列表，使用逗号分隔元素：`[a]`，`[a,b,c]`
* 使用列表解析（list comprehension）：`[x for x in iterable]`
* 使用类型构造函数（type constructor）：`list()`或`list(iterable)`

使用构造函数方式创建的列表，它的元素以及元素的顺序跟iterable中的元素完全一样。如果iterable本身就是列表，那么返回的是iterable的副本，相当于`iterable[:]`。例如`list('abc')`返回`['a', 'b', 'c']`；`list( (1, 2, 3) )`返回`[1, 2, 3]`。

#### `sort(*, key=None, reverse=None)`

列表支持序列的通用操作和可变序列的操作。除此之外，列表还提供了以下方法：`sort(*, key=None, reverse=None)`。该方法在原列表上进行排序，对元素使用小于比较（`<`）。`sort()`方法接受两个只能通过参数，关键字传递的参数。

`key`指定一个接受单个参数的函数，用于为每个列表元素提取比较关键字（comparison key），例如`key=str.lower`。参数`key`默认值为`None`，表示不计算比较关键字直接排序。

`reverse`是一个布尔值。如果设置为`True`，那么就倒序排序。

`sort()`方法为了节省空间，直接在原列表上进行排序。该方法并不返回排序后的列表（使用`sorted()`方法来获取新的排序后的列表）。

#### 列表生成式

列表生成式（List Comprehensions）是在一个序列的值上应用一个表达式，将计算结果放到一个新的列表中并返回。

* `[expr for iter_var in iterable]`：迭代iterable里所有元素，每一次迭代时把iterable里相应元素放到iter_var中，再在表达式中应用该iter_var的内容，最后用表达式的计算值生成一个列表。
* `[expr for iter_var in iterable if cond_expr]`：加入了判断语句，只有满足条件的内容才把iterable里相应内容放到iter_var中，再在表达式中应用该iter_var的内容，最后用表达式的计算值生成一个列表。

```python
>>> [x*2 for x in range(10)]
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
>>> [x*2 for x in range(10) if x%2==0]
[0, 4, 8, 12, 16]
>>> [(x,y) for x in range(5) if x%2==0 for y in range(5) if y%2==1]
[(0, 1), (0, 3), (2, 1), (2, 3), (4, 1), (4, 3)]
```

TODO：跟列表生成式相似的有生成器表达式。


### 元组（Tuple）

元组是不可变序列，是只读版的列表。

#### 元组构建方式

元组的创建方式有以下几种：
* 空元组：`()`
* 单个元素的元组（独元）：`a,`或`(a,)`
* 多个元素使用逗号分隔：`a,b,c`或`(a,b,c)`
* 使用类型构造函数（：`tuple()`或`tuple(iterable)`

使用构造函数方式创建的元组，它的元素以及元素的顺序跟iterable中的元素完全一样。如果iterable本身就是元组，那么返回的是完全一样的。例如`tuple('abc')`返回`('a', 'b', 'c')`；`tuple( (1, 2, 3) )`返回`(1, 2, 3)`。

注意，是逗号的作用创建的元组，而不是括号。括号可以省略的，除了空元组和避免语法歧义时。例如`f(a, b, c)`是一个有三个参数的函数调用，而`f((a, b, c))`是一个单个参数为三元素元组的函数调用。

元组支持所有序列的通用操作。

具名元组`collections.namedtuple()`可以使用名称来代替索引访问元素。在某些场合可能比简单元素更好用。

#### 元组中元素的可变与不可变

元组本身是不可变的，这意味着：如果元组为t，可以通过t[i]来得到第i个成员的值，但无法通过t[i]去修改第i个成员的值。

元组包含的成员是否可变，与元组的不可变无关，只取决于成员本身的可变性。例如元组t的第i个成员是一个列表l，则可以通过t[i][j]去修改列表l的第j个成员的值。

```python
>>> t=(1,[2,3,4],(5,6),'7','8')
>>> t[0]=9
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'tuple' object does not support item assignment
>>> t[1][1]=10
>>> t
(1, [2, 10, 4], (5, 6), '7', '8')
>>> t[2][1]=11
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'tuple' object does not support item assignment
```

### Range

Range是不可变的数字序列，通常用于for循环。

* `range(stop)`
* `class range(start, stop[, step])`

range构造函数的参数必须为整数。如果参数`step`没有指定，那么为默认值1。如果参数`start`没有指定，那么为默认值0。如果`step`为0,那么会引发ValueError异常。

如果`step`为正数，那么range r的内容为：`r[i] = start + step*i`，当`i >= 0`且`r[i] < stop`。

如果`step`为负数，那么range r的内容为：`r[i] = start + step*i`，当`i >= 0`且`r[i] > stop`。

```python
>>> list(range(10))
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
>>> list(range(1, 11))
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
>>> list(range(0, 30, 5))
[0, 5, 10, 15, 20, 25]
>>> list(range(0, 10, 3))
[0, 3, 6, 9]
>>> list(range(0, -10, -1))
[0, -1, -2, -3, -4, -5, -6, -7, -8, -9]
>>> list(range(0))
[]
>>> list(range(1, 0))
[]
```

range支持所用通用序列的操作，除了串联（concatenation）和重复(repetition)。

range相对于列表和元组的优势在于，它占用的内存很小，而且是固定大小的。它只存储start，stop和step这三个值。

range支持存在性测试（containment test）、元素索引查找、切片和负数索引：

```python
>>> r = range(0, 20, 2)
>>> r
range(0, 20, 2)
>>> 11 in r
False
>>> 10 in r
True
>>> r.index(10)
5
>>> r[5]
10
>>> r[:5]
range(0, 10, 2)
>>> r[-1]
18
```

两个range的相等性判断跟序列是一样的。如果两个range对象所代表的序列值是一样的，那么它俩就是相等的。

```python
>>> range(0) == range(2, 1, 3)
True
>>> range(0, 3, 2) == range(0, 4, 2)
True
```

---

参考资料：

[Python3 Documentation](https://docs.python.org/3.5/library/stdtypes.html#sequence-types-list-tuple-range)
