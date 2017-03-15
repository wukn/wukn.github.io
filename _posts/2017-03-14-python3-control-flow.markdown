---
layout:     post
title:      "[Python3] 控制流"
subtitle:   ""
date:       2017-03-14 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Python3

---

> Python3 控制流。

### while

while语句用于在表达式正确的条件下重复执行。

它重复测试表达式，如果为真，则执行第一个语句组；如果表达式为假（可能是第一次测试），则执行else子句且终止循环。

第一个语句组中执行的break语句会终止循环而不执行else子句的语句组。在第一个语句组中执行的continue语句会跳过语句组中剩余的语句并返回继续测试表达式。

```python
n=0
while n<5:
    print(n)
else
    print('err')
```

### if

if语句用于条件判断。

```python
if x >= 90:
    print('优秀')
elif x >= 80:
    print('良好')
elif x >= 70:
    print('中等')
elif x >= 60:
    print('及格')
else:
    print('不及格')
```

可以有零个或多个elif部分，else部分是也可选的。if... elif ... elif ... 序列用于替代其它语言中switch或case语句。

### for

Python的for语句可以循环访问任何序列（列表或一个字符串），按照它们在序列中出现的顺序。

```python
words = ['cat', 'window', 'defenestrate']
for w in words:
    print(w, len(w))

# output like this:
# cat 3
# window 6
# defenestrate 12
```

如果要在循环体内修改正在迭代的序列（例如复制所选的项目），建议首先制作副本。迭代序列不会隐式地创建副本。 可以使用切片来创建副本：

```python
words = ['cat', 'window', 'defenestrate']
for w in words[:]:  # Loop over a slice copy of the entire list.
    if len(w) > 6:
      words.insert(0, w)

# words
# ['defenestrate', 'cat', 'window', 'defenestrate']
```

通常使用`range()`函数生成一个等差的数字序列用于遍历。例如根据索引来迭代序列：

```python
a = ['Mary', 'had', 'a', 'little', 'lamb']
for i in range(len(a)):
    print(i, a[i])

# output：
# 0 Mary
# 1 had
# 2 a
# 3 little
# 4 lamb
```

其实在这种情况下，使用`enumerate()`函数会更加方便。

`enumerate(iterable, start=0)`：返回一个枚举对象。iterable必须是序列、迭代器，或者其他支持迭代的对象。由`enumerate()`返回的迭代器的__next__()方法返回一个元组，该元组包括一个计数器（从 start 开始，start默认值为0）和对iterable的迭代取得的值。

```python
>>> seasons = ['Spring', 'Summer', 'Fall', 'Winter']
>>> list(enumerate(seasons))
[(0, 'Spring'), (1, 'Summer'), (2, 'Fall'), (3, 'Winter')]
>>> list(enumerate(seasons, start=1))
[(1, 'Spring'), (2, 'Summer'), (3, 'Fall'), (4, 'Winter')]
```

相当于：

```python
def enumerate(sequence, start=0):
    n = start
    for elem in sequence:
        yield n, elem
        n += 1
```

当我们打印range时，会发现结果并不是我们想的是一个列表：

```python
>>> print(range(10))
range(0, 10)
```

range()返回的对象的行为在很多方面很像一个列表，但实际上它并不是列表。当你迭代它的时候它会依次返回期望序列的元素，但是它不会真正产生一个列表，因此可以节省空间。
这样的对象称为可迭代对象（iterable）。它们适合被一些函数和构造器使用，这些函数和构造器期望可以从可迭代对象中获取连续的元素，直到没有新的元素为止。for语句就是这样的一个迭代器（iterator）。还有一个是`list()`函数，它从可迭代对象创建列表。

```python
>>> list(range(5))
[0, 1, 2, 3, 4]
```

### break、continue和循环中的else子句

break语句用于跳出最近的for或while循环。

break在语法上只可以出现在for或者while循环中，但不能嵌套在这些循环内的函数和类定义中。

它终止最相近的循环，如果循环有else子句将跳过。

如果break终止了一个for循环，控制循环的目标保持当前的值。

当break将控制传出带有finally子句的try语句时，在离开循环之前会执行finally子句。

循环语句可以有一个else子句；当循环是因为迭代完整个列表(for语句)或者循环条件不成立（while语句）终止，而非由break语句终止时，else子句将被执行。

```python
for n in range(2, 10):
    for x in range(2, n):
        if n % x == 0:
            print(n, 'equals', x, '*', n//x)
            break
    else:
        # loop fell through without finding a factor
        print(n, 'is a prime number')

# output：
2 is a prime number
3 is a prime number
4 equals 2 * 2
5 is a prime number
6 equals 2 * 3
7 is a prime number
8 equals 2 * 4
9 equals 3 * 3
```

仔细看，上面的代码中else子句属于for循环，不属于if语句。

与循环一起使用的else子句更类似于try语句的else子句而不是if语句的else子句：try语句的else子句在没有任何异常发生时运行，而循环的else子句在没有break发生时运行。

continue语句表示继续下一次迭代：

```python
for num in range(2, 10):
    if num % 2 == 0:
        print("Found an even number", num)
        continue
    print("Found a number", num)

# output
# Found an even number 2
# Found a number 3
# Found an even number 4
# Found a number 5
# Found an even number 6
# Found a number 7
# Found an even number 8
# Found a number 9
```

continue在语法上只可以出现在for或while循环中，但不能嵌套在这些循环内的函数定义、类定义和finally子句中。它继续最内层循环的下一轮。

当continue将控制传出带有finally子句的try语句时，在真正开始下一轮循环之前会执行finally子句。

### pass

pass语句什么也不做。它用于语法上必须要有一条语句，但程序什么也不需要做的场合。通常在编写新代码时作为函数体或控制体的占位符。

它也通常用于创建最小的类：

```python
class MyEmptyClass:
    pass
```

---

参考资料：

[Python3 Documentation - Tutorial](https://docs.python.org/3.5/tutorial/controlflow.html
