---
layout:     post
title:      "[Python3] 数值"
subtitle:   ""
date:       2017-03-08 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Python3

---

> Python3 数值。

Python中的数值分为整型（int）、浮点型（float）和复数（complex）。还有一个布尔型（bool）是整型的子类型。

数值类型支持的主要操作如下：

![](/img/post/python3/numbers/python3-numbers.png)

#### 布尔型（bool）

布尔型是整型的子类型。布尔值有两个：True和False。布尔值的行为在几乎所有环境下分别类似0和1，例外的情况是转换成字符串的时候分别返回字符串"False" 或"True"。

任何对象都可以做真值检测。以下对象的布尔值都是False：

* None
* False（布尔型）
* 任何等于0的数字类型, 比如, 0，0.0，0j
* 任何空的序列, 比如, 空字符串'', 空列表(), 空元组[]
* 任何空的映射, 比如, 空字典{}
* 自定义的类实例，该类定义了方法`__nonzero__()`或`__len__()`，并且这个方法返回0或False

```python
>>> bool(None)
False
>>> bool(False), bool(0), bool(0.0), bool(0.0+0.0j)
(False, False, False, False)
>>> bool(''), bool([]), bool(()), bool({})
(False, False, False, False)
>>>
>>> int(True), int(False)
(1, 0)
```

以下是Boolean操作, 按照优先级升序排列：

* （1） 或操作：`x or y`
* （2） 与操作：`x and y`
* （3） 非操作：`not x`

PS1：或操作和与操作都是短路求值运算符。
PS2：非操作比非布尔运算符的优先级更低，因此`not a == b`被解释成`not (a == b)`, 而`a == not b`是一个语法错误

#### 整形

整型可以表示无限大的整数（但受限于机器的虚拟内存大小）。整型字面值的表示方法有3种：十进制（常用）、八进制（以“0o”或“0O”开头）和十六进制（以“0x”或“0X”开头）。

```python
>>> 1/2
0.5
>>> 1//2
0
>>> 2**3
8
>>> bin(20), oct(20), hex(20)
('0b10100', '0o24', '0x14')
>>> 322 ** 10
11982811207963436516967424
```

#### 浮点型

浮点型字面值可以用十进制或科学计数法表示，在科学计数法中，e或E代表10，+（可以省略）或 - 表示指数的正负。

```python
>>> 2.0 / 1e-1
20.0
>>> 2.0 / 1E-1
20.0
>>> 2.0 ** 3.0 //3
2.0
>>> round(2.71828, 3)
2.718
>>> round(-2.71828, 3)
-2.718
```

#### 复数

复数由实数部分和虚数部分构成，表示为eal+imagj 或 real+imagJ。实部real和虚部imag都是浮点型。

```python
>>> x=-1.2 -3.4j
>>> x
(-1.2-3.4j)
>>> x.real
-1.2
>>> x.imag
-3.4
>>> x.conjugate() # 共轭复数
(-1.2+3.4j)
>>> abs(x) # math.sqrt(c.real ** 2 + c.imag ** 2)
3.605551275463989
```

#### 类型转换

如果参与运算的两个操作数的类型不同，Python会按照以下规则进行自动类型转换：
* 如果有一个操作数是复数，另一个操作数被转换为复数
* 否则，如果有一个操作数是浮点型，另一个操作数被转换为浮点型
* 否则，两者必然都是整型，无须类型转换
总结一下就是：非复数转复数，非浮点型转浮点型，整型不变。

```python
>>> 1.0 + (2+4j)
(3+4j)
>>> 3 + 4.56
7.56
>>> 7 + 8
15
>>> 9 + True
10
```

也可以强制类型转换：

```python
>>> bool(5.0)
True
>>> int(5.0)
5
>>> long(5.0)
5L
>>> float(5)
5.0
>>> complex(1, 2.3)
(1+2.3j)
```

#### 相关模块

Python标准库中与数值类型相关的核心模块有：

* numbers：数字型抽象基类
* math和cmath：标准C库数学运算函数。math模块为常规运算，cmath模块为复数运算
* decimal：十进制浮点小数运算
* fractions：分数
* random：生成伪随机数
* statistics：数学统计函数

```python
>>> x=1.23
>>> import math
>>> x
1.23
>>> math.trunc(x) # 将x截断为整型
1
>>> math.floor(x) # 小于等于x的最大的整数（向下取整）
1
>>> math.ceil(x) # 大于等于x的最小的整数（向上取整）
2
```

---

参考资料：

[Python3 Documentation](https://docs.python.org/3.5/)
