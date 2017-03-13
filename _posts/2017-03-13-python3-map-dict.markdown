---
layout:     post
title:      "[Python3] 映射（字典）"
subtitle:   ""
date:       2017-03-13 00:00:00
author:     "wukn"
header-img: ""
catalog: true
tags:
    - Python3

---

> Python3 映射——字典。

### 映射类型（Mapping Type）

一个映射对象将可哈希的值与任意对象建立映射关系。映射是可变对象。目前只有一种标准映射类型，字典（dictionary）。

与序列不同，序列由数字做索引，字典由键（key）做索引，键可以是任意不可变类型（可哈希的）；字符串和数字永远可以拿来做键。如果元组只包含字符串、 数字或元组，它们可以用作键；如果元组直接或间接地包含任何可变对象，不能用作键。不能用列表作为键，因为列表可以用索引、切片或者append()和extend()方法修改。

另外，不建议使用浮点数作为key，因为计算机存储的是浮点数的近似值。

理解字典的最佳方式是把它看做无序的 键:值 对集合，要求是键必须是唯一的（在同一个字典内）。一对大括号将创建一个空的字典：`{}`。大括号中由逗号分隔的 键:值 对将成为字典的初始值。例如，`'tom': 4098, 'jerry': 4127}`、`{4098: 'tom', 4127: 'jerry'}`。

还可以使用dict构造函数创建字典：

* `dict(**kwarg)`
* `dict(mapping, **kwarg)`
* `dict(iterable, **kwarg)`

根据一个可选的位置参数和一个可能为空集合的关键字参数初始化一个字典并返回。

如果没有给位置参数，那么创建的是空字典。如果位置参数是一个映射对象，那么创建的字典的键值对跟这个映射对象是相同的。另外，位置参数也可能是个迭代对象（iterable object），iterable内的每个元素必须是正好包含两个对象的迭代对象，元素的第一个对象作为字典的键，第二个对象作为字典的值。如果一个键出现多次，那么最后出现的对应的值将成为字典中这个键对应的值。

如果指定了关键字参数，关键字参数和它们的值将添加到由位置参数创建的字典中，如果关键字参数中出现已经存在的键，那么它对应的值将覆盖之前位置参数中的值。

```python
>>> a = dict(one=1, two=2, three=3)
>>> b = {'one': 1, 'two': 2, 'three': 3}
>>> c = dict(zip(['one', 'two', 'three'], [1, 2, 3]))
>>> d = dict([('two', 2), ('one', 1), ('three', 3)])
>>> e = dict({'three': 3, 'one': 1, 'two': 2})
>>> a == b == c == d == e
True
```

### 字典操作

字典支持的操作如下：

* `len(d)`：返回字典d中元素的数量。
* `d[key]`：返回字典d中键key对应的元素。如果映射中没有key会引发KeyError异常。如果字典的子类定义了`__missing__()`方法，那么当key不存在时，d[key]操作会调用这个方法，将key作为参数传递过去。
* `d[key] = value`：将d[key]的值设置为value。
* `del d[key]`：删除字典d中的d[key]。如果映射中没有key会引发KeyError异常。
* `key in d`：如果字典d中有key则返回True，否则返回False。
* `key not in d`和`not key in d`：如果字典d中没有key则返回True，否则返回False。
* `iter(d)`：返回字典d的所有键的迭代器。这是简版的`iter(d.keys())`。
* `clear()`：移除字典中的所有元素。
* `copy()`：返回字典的一个浅拷贝。
* `fromkeys(seq[, value])`：这是一个类方法。创建一个新字典，它的键从seq中取，值为value。参数value默认为None。
* `get(key[, default])`：如果字典中存在键key，那么返回对应的值，否则返回default。参数default没有指定时则默认为None。这个方法是为了避免引发KeyError异常。
* `items()`：返回字典项（`(key, value)`对）的视图。
* `keys()`：返回字典键的视图。
* `pop(key[, default])`：如果字典中存在键key，那么移除它并返回它对应的值，否则返回default。如果映射中没有key会引发KeyError异常。
* `popitem()`：移除并返回字典中任意一个键值对。如果字典为空会引发KeyError异常。
* `setdefault(key[, default])`：如果字典中存在键key，那么返回它对应的值。否则，插入(key,default)，并返回default。参数default默认为None。
* `update([other])`：使用other中的键值对来更新当前字典，覆盖已存在的键。返回None。参数other可以是字典，也可以是键值对迭代器，或者是关键字参数。
* `values()`：返回字典值的视图。
* `list(d.keys())`：返回字典中所有键组成的列表，列表的顺序是随机的（如果希望它是有序的，只需使用sorted(d.keys())代替）。

### 字典视图对象（Dictionary View Object）

` dict.keys()`、`dict.values()`和`dict.items()`返回的对象叫做试图对象（View Object）。它们提供字典的动态试图，也就是说字典发生变化时，视图会实时反映这些变化。

字典视图可以被yield迭代，并支持成员测试。

* `len(dictview)`：返回字典的长度。
* `iter(dictview)`：返回键、值或元素（相当于元组(key, value)）的迭代器。
键和值的迭代顺序是任意的，但并不是随机的。迭代顺序根据Python的实现方式而异，而且依赖字典的插入和删除历史操作。如果字典没有处于intervening modification状态，那么它的键视图、值视图和元素视图中元素的顺序是一致的。也就是说可以使用`zip()`来创建键值对：`pairs = zip(d.values(), d.keys())`，另一种方法是`pairs = [(v, k) for (k, v) in d.items()]`。
* `x in dictview`：如果视图对应的字典的键、值或元素存在x，则返回True。

键视图跟集合很像，因为它的条目是唯一且可哈希的。如果所有值都是可哈希的，那么元素视图也是跟集合相似的。值视图由于很可能有重复的，它并不被当成集合相似的。对于集合相似的视图，抽象基类collections.abc.Set定义的操作都支持，例如`==`、`<`或`^`。

```python
>>> dishes = {'eggs': 2, 'sausage': 1, 'bacon': 1, 'spam': 500}
>>> keys = dishes.keys()
>>> values = dishes.values()

>>> # iteration
>>> n = 0
>>> for val in values:
...     n += val
>>> print(n)
504

>>> # keys and values are iterated over in the same order
>>> list(keys)
['eggs', 'bacon', 'sausage', 'spam']
>>> list(values)
[2, 1, 1, 500]

>>> # view objects are dynamic and reflect dict changes
>>> del dishes['eggs']
>>> del dishes['sausage']
>>> list(keys)
['spam', 'bacon']

>>> # set operations
>>> keys & {'eggs', 'bacon', 'salad'}
{'bacon'}
>>> keys ^ {'sausage', 'juice'}
{'juice', 'sausage', 'bacon', 'spam'}
```

### 稀疏矩阵

使用元组作为字典的键，可以构建类似稀疏矩阵的数据结构：

```python
>>> matrix = {}
>>> matrix[(2, 3, 4)] = 88
>>> matrix[(7, 8, 9)] = 99
>>> matrix
{(2, 3, 4): 88, (7, 8, 9): 99}
>>> x, y, z = 2, 3, 4
>>> matrix[(x, y, z)]
88
```

---

参考资料：

[Python3 Documentation - Language Reference](https://docs.python.org/3.5/library/stdtypes.html#dict)

[Python3 Documentation - Tutorial](https://docs.python.org/3.5/tutorial/datastructures.html#dictionaries)
