---
layout:     post
title:      "[Python3] 函数"
subtitle:   ""
date:       2017-03-15 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Python3

---

> Python3 函数。

### 定义函数

函数是一个可调用的对象，它接收0个或多个参数，然后执行一段代码，最后返回一个值给调用者。

在python中函数是first-class object。因此它具有与其他Python对象完全相同的基本行为特征，如可以被传递、可以作为右值进行赋值、可以作为另一个函数的参数或返回值等。

```python
def fib(n):    # write Fibonacci series up to n
    """Print a Fibonacci series up to n."""
    a, b = 0, 1
    while a < n:
        print(a, end=' ')
        a, b = b, a+b
    print()
```

```python
>>> fib(2000)
0 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987 1597
```

关键字def用于定义函数。其后必须跟有函数名称和以括号标明的形式参数（formal parameter）列表。函数体语句从下一行开始，且必须缩进。

函数体的第一行可以是一个可选的字符串文本；该字符串是函数的文档字符串（docstring）。

函数执行时会引入一个新的符号表，用于保存函数的局部变量。更确切地说，函数中的所有的赋值都是将值存储在局部符号表；而变量引用首先查找局部符号表，然后是上层函数的局部符号表，然后是全局符号表，最后是内置名字表。因此，在函数内部虽然可以引用全局变量，但是却不能直接给它们赋值（除非用global语句命名）。

被调函数的实际参数是在被调用时从其局部符号表中引入的；因此，参数的传递使用传值调用（这里的值始终是对象的引用，不是指对象的值）。事实上，按对象引用传递可能是更恰当的说法，因为如果传递了一个可变对象，调用函数将看到任何被调用函数对该可变对象做出的改变（比如添加到列表中的元素）。

函数定义会在当前符号表内引入函数名。函数名对应的值的类型是解释器可识别的用户自定义函数。此值可以分配给另一个名称，然后该名称也可作为函数。

```python
>>> fib
<function fib at 10042ed0>
>>> f = fib
>>> f(100)
0 1 1 2 3 5 8 13 21 34 55 89
```

没有return语句的函数也会返回一个值，这个值为None。

```python
>>> fib(0)
>>> print(fib(0))
None
```

将函数改成返回Fibonacci数列列表：

```python
def fib2(n): # return Fibonacci series up to n
    """Return a list containing the Fibonacci series up to n."""
    result = []
    a, b = 0, 1
    while a < n:
        result.append(a)    # see below
        a, b = b, a+b
    return result

```

```python
>>> f100 = fib2(100)    # call it
>>> f100                # write the result
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]
```

### 参数默认值

创建函数时，可以指定一个或多个参数的默认值。这样的函数被调用时，可以省略带有默认值的参数。

```python
def ask_ok(prompt, retries=4, complaint='Yes or no, please!'):
    while True:
        ok = input(prompt)
        if ok in ('y', 'ye', 'yes'):
            return True
        if ok in ('n', 'no', 'nop', 'nope'):
            return False
        retries = retries - 1
        if retries < 0:
            raise OSError('uncooperative user')
        print(complaint)
```

这个函数可以有几种调用方式：
* 只给出强制参数：`ask_ok(' Do you really want to quit?')`
* 给出一个可选的参数：`ask_ok('OK to overwrite the file?', 2)`
* 给出所有的参数：`ask_ok('OK to overwrite the file?', 2, 'Come on, only yes or no!')`

上面的代码中还使用了in关键字，它用于测试一个序列是否包含特定的值。

参数的默认值是在函数的定义阶段计算的，因此：

```python
i = 5

def f(arg=i):
    print(arg)

i = 6
f()

# will output：
# 5
```

注意，参数的默认值只计算一次，但是当默认值是可变对象时又不一样，例如列表、字典或大部分类的实例。

```python
def f(a, L=[]):
    L.append(a)
    return L

print(f(1))
print(f(2))
print(f(3))

# will output:
# [1]
# [1, 2]
# [1, 2, 3]
```

如果不希望默认值在随后的调用中共享，可以这样做：

```python
def f(a, L=None):
    if L is None:
        L = []
    L.append(a)
    return L
```

### 关键字参数

使用位置参数（positional argument）调用函数时，实参根据顺序和形参对应。

除了位置参数，函数也可以通过`kwarg = value`这样关键字参数（keyword argument）的形式调用。

```python
def parrot(voltage, state='a stiff', action='voom', type='Norwegian Blue'):
    print("-- This parrot wouldn't", action, end=' ')
    print("if you put", voltage, "volts through it.")
    print("-- Lovely plumage, the", type)
    print("-- It's", state, "!")
```

上面的函数接受一个必选参数和三个可选参数。我们可以这样调用它：

```python
parrot(1000)                                          # 1 positional argument
parrot(voltage=1000)                                  # 1 keyword argument
parrot(voltage=1000000, action='VOOOOOM')             # 2 keyword arguments
parrot(action='VOOOOOM', voltage=1000000)             # 2 keyword arguments
parrot('a million', 'bereft of life', 'jump')         # 3 positional arguments
parrot('a thousand', state='pushing up the daisies')  # 1 positional, 1 keyword
```

但不能这样调用：

```python
parrot()                     # required argument missing
parrot(voltage=5.0, 'dead')  # non-keyword argument after a keyword argument
parrot(110, voltage=220)     # duplicate value for the same argument
parrot(actor='John Cleese')  # unknown keyword argument
```

在调用函数时，关键字参数必须在位置参数后面，关键字参数名必须和函数的某个参数匹配，他们的顺序并不重要。任何参数都不能多次赋值。

如果最后一个参数以`**name`的形式出现，那么它可以接收一个字典。该字典包含了所有未出现在形式参数列表中的关键字参数。它还可能与`*name`这样的形参结合使用。`*name`接收一个元组，该元组包含了所有未出现在形参列表中的位置参数。`*name`必须出现在`**name`之前。

```python
def cheeseshop(kind, *arguments, **keywords):
    print("-- Do you have any", kind, "?")
    print("-- I'm sorry, we're all out of", kind)
    for arg in arguments:
        print(arg)
    print("-" * 40)
    keys = sorted(keywords.keys())
    for kw in keys:
        print(kw, ":", keywords[kw])
```

它可以这样被调用：

```python
cheeseshop("Limburger", "It's very runny, sir.",
           "It's really very, VERY runny, sir.",
           shopkeeper="Michael Palin",
           client="John Cleese",
           sketch="Cheese Shop Sketch")
```

输出的结果为：

```python
-- Do you have any Limburger ?
-- I'm sorry, we're all out of Limburger
It's very runny, sir.
It's really very, VERY runny, sir.
----------------------------------------
client : John Cleese
shopkeeper : Michael Palin
sketch : Cheese Shop Sketch
```

### 任意参数列表

还有一种不太常见的场景是使用可变数量的参数调用函数。这些参数将会被放到一个元组中。在可变数量参数之前，可以有零个或多个普通的参数。

```python
def write_multiple_items(file, separator, *args):
    file.write(separator.join(args))
```

通常可变数量参数位于形参列表的最后，因为它会收集传递过来的所有剩余的输入参数。出现在`*args`参数后面的任何形参只能作为关键字参数。

```python
>>> def concat(*args, sep="/"):
...    return sep.join(args)
...
>>> concat("earth", "mars", "venus")
'earth/mars/venus'
>>> concat("earth", "mars", "venus", sep=".")
'earth.mars.venus'
```

### 参数列表解包

如果传递的参数已经是一个列表或元组，那么我们将它解包，拆分成独立的位置参数。例如，`range()`函数期望两个单独的参数start和stop，如果它们不是独立的（放在一个列表或元组中），那么函数调用时得使用`*`操作符将参数从列表或元组中分拆开来：

```python
>>> list(range(3, 6))            # normal call with separate arguments
[3, 4, 5]
>>> args = [3, 6]
>>> list(range(*args))            # call with arguments unpacked from a list
[3, 4, 5]
```

同样，可以使用`**`操作符让字典传递关键字参数：

```python
>>> def parrot(voltage, state='a stiff', action='voom'):
...     print("-- This parrot wouldn't", action, end=' ')
...     print("if you put", voltage, "volts through it.", end=' ')
...     print("E's", state, "!")
...
>>> d = {"voltage": "four million", "state": "bleedin' demised", "action": "VOOM"}
>>> parrot(**d)
-- This parrot wouldn't VOOM if you put four million volts through it. E's bleedin' demised !
```

### lambda表达式

可以使用lambda关键字创建匿名函数。在语法上，它们被限制只能有一个单独的表达式。在语义上，他们只是普通函数定义的语法糖。就像嵌套的函数定义，lambda函数可以从它被包含的作用域中引用变量：

```python
>>> def make_incrementor(n):
...     return lambda x: x + n
...
>>> f = make_incrementor(42)
>>> f(0)
42
>>> f(1)
43
```

上面的代码中是使用lambda表达式返回一个函数。另一种用法是将一个函数作为参数传递。

```python
>>> pairs = [(1, 'one'), (2, 'two'), (3, 'three'), (4, 'four')]
>>> pairs.sort(key=lambda pair: pair[1])
>>> pairs
[(4, 'four'), (1, 'one'), (3, 'three'), (2, 'two')]
```

### 文档字符串（docstring）

文档字符串用于对对象进行注释。它的格式的惯例为：

第一行永远应该是对象用途的简短、精确的总述。为了简单起见，不需要明确的陈述对象的名字或类型，因为这些信息可以从别的途径了解到。

如果文档字符串中还要有其他描述，那么第二行应该是空白，在视觉上把摘要与剩余的描述分开。接下来应该是对对象的调用约定、 其副作用等的描述。

```python
>>> def my_function():
...     """Do nothing, but document it.
...
...     No, really, it doesn't do anything.
...     """
...     pass
...
>>> print(my_function.__doc__)
Do nothing, but document it.

    No, really, it doesn't do anything.
```

### 函数注解（Function Annotations）

函数注解是函数的一些元数据信息，用于描述自定义函数中的类型。

注解以字典的形式存储在函数的__annotations__属性中。参数的注释用参数名后的冒号定义,冒号后面紧跟着一个用于计算注释的表达式。回值注释使用`->`定义，紧跟着参数列表和def语句的末尾的冒号之间的表达式。

```python
>>> def f(ham: str, eggs: str = 'eggs') -> str:
...     print("Annotations:", f.__annotations__)
...     print("Arguments:", ham, eggs)
...     return ham + ' and ' + eggs
...
>>> f('spam')
Annotations: {'ham': <class 'str'>, 'return': <class 'str'>, 'eggs': <class 'str'>}
Arguments: spam eggs
'spam and eggs'
```

### （附）编码风格

PEP 8是大部分Pyhton项目所遵循的风格指南。其要点有：

* 使用4个空格的缩进，不要使用制表符。4个空格是小缩进（允许更深的嵌套）和大缩进（易于阅读）之间很好的折衷。制表符会引起混乱，最好弃用。
* 每一行以确保不会超过79个字符。这有助于小显示器用户阅读，也可以让大显示器能并排显示几个代码文件。
* 使用空行分隔函数和类，以及函数内的大块代码。
* 如果可能，注释独占一行。
* 使用文档字符串。
* 运算符周围和逗号后使用空格，但是括号里侧不加空格：`a = f(1, 2) + g(3, 4)`。
* 保持类和函数命名的一致性；常见的做法是命名类的时候使用驼峰法，命名函数和方法的时候使用小写字母+下划线法。始终使用self作为第一个方法参数的名称。
* 尽量只使用Python默认的UTF-8编码。

---

参考资料：

[Python3 Documentation - Tutorial](https://docs.python.org/3.5/tutorial/controlflow.html#defining-functions)
