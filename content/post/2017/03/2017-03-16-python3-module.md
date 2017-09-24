---
author: wukn
catalog: true
date: 2017-03-16T00:00:00Z
tags:
- Python3
title: '[译][Python3] 模块'
url: /2017/03/16/python3-module/
---

> Python3 模块。

<!--more-->

### 模块（Ｍodule）

通常我们将代码写到文件中（所谓的创建脚本），然后运行这个文件。对于很长的程序，我们希望将它分成多个文件，这样更易于维护和复用。

Python支持将程序的内容放在文件中，然后在其它脚本或交互方式中使用。这种文件就成为模块（Module）。

模块是包含Python定义和声明的文件。文件名就是模块名加上.py后缀。在模块里面，模块名（是一个字符串）可以通过访问全局变量`__name__`的值得到。

例如，创建一个`fibo.py`文件：

```python
# Fibonacci numbers module

def fib(n):    # write Fibonacci series up to n
    a, b = 0, 1
    while b < n:
        print(b, end=' ')
        a, b = b, a+b
    print()

def fib2(n): # return Fibonacci series up to n
    result = []
    a, b = 0, 1
    while b < n:
        result.append(b)
        a, b = b, a+b
    return result
```

现在可以在Python解释器中导入并使用这个模块：

```python
>>> import fibo
```

这样只是把模块名字`fibo`导入当前的符号表中，而不是把`fibo`中的函数名字导入其中。我们可以通过模块名访问这些函数：

```python
>>> fibo.fib(1000)
1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987
>>> fibo.fib2(100)
[1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]
>>> fibo.__name__
'fibo'
```

对于频繁使用的函数，可以把他赋给一个本地变量：

```python
>>> fib = fibo.fib
>>> fib(500)
1 1 2 3 5 8 13 21 34 55 89 144 233 377
```

import模块或模块的内容时，可以起一个别名，然后使用别名来访问它：

```python
>>> import fibo as fibonacci
>>> fibonacci.fib(500)
1 1 2 3 5 8 13 21 34 55 89 144 233 377
```

### 模块的导入

模块除了可以包含定义的函数，还可以包含可执行语句。这些语句通常是为了初始化模块，它们只在模块第一次导入时执行。（如果文件作为脚本运行，那它们也会执行。）

每个模块都有它自己的似有符号表。模块内的函数使用它作为全局符号表。因此，在模块内使用全局变量不会与其它模块的全局变量有冲突。另外，可以使用引用模块函数的表示法访问模块的全局变量：`modname.itemname`。

模块中可以导入其它模块。习惯上将import语句放在模块（或脚本）的开始。被导入的模块的名字放在导入模块的全局符号表中。

另一种import语句可以从被导入模块中将名字导入到模块的符号表中，这样并不会把模块名导入到本地的符号表中。

```python
>>> from fibo import fib, fib2
>>> fib(500)
1 1 2 3 5 8 13 21 34 55 89 144 233 377
```

或者导入所有名字：

```python
>>> from fibo import *
>>> fib(500)
1 1 2 3 5 8 13 21 34 55 89 144 233 377
```

这种方式不会导入以下划线 ( _ ) 开头的名称。通常不建议这样做，因为这样会使代码的可读性变差。

### 将模块作为脚本执行

当使用下列方式运行一个Python模块：

```bash
python fibo.py <arguments>
```

模块中的代码将会被执行，就像导入它一样，只不过这时候`__name__`被设置为`__main__`。因此，如果在模块里加入如下代码：

```python
if __name__ == "__main__":
    import sys
    fib(int(sys.argv[1]))
```

就可以让这个文件既作为可执行脚本，也可以作为模块被导入。因为解析命令行的那部分代码只有在模块作为 “main” 文件执行时才被调用。

```bash
$ python fibo.py 50
1 1 2 3 5 8 13 21 34
```

而导入该模块时，则不会运行这段代码：

```python
>>> import fibo
>>>
```

通常可以使用这种方式为模块提供用户接口或用于运行测试用例。

### 模块搜索路径

当导入一个名为spam的模块时，解释器首先在内置模块中搜索。如果没有找到，它会接着到sys.path变量给出的目录中查找名为spam.py的文件。sys.path变量的初始值来自这些位置：
* 脚本所在的目录（如果没有指明文件，则为当前目录）
* PYTHONPATH（一个包含目录名的列表，跟shell变量PATH的语法相同）
* 与安装相关的默认值

注意：在支持符号链接的文件系统中，输入的脚本所在的目录是符号链接所指向的目录。也就是说包含符号链接的目录不会被加到目录搜索路径中。

在初始化之后，Python程序可以修改sys.path。脚本所在的目录被放置在搜索路径的最开始，也就是在标准库的路径之前。这意味着将会加载当前目录中的脚本，库目录中具有相同名称的模块不会被加载。

### 标准模块

Python带有一个标准模块库，详细可见[库参考手册](https://docs.python.org/3/library/index.html)。

有一个特别的模块sys，它内置在每一个Python解释器中。

变量sys.path是一个字符串列表，它决定了解释器搜索模块的路径。它初始的默认路径来自于环境变量PYTHONPATH，如果PYTHONPATH未设置则来自于内置的默认值。可以使用标准的列表操作修改它：

```python
>>> import sys
>>> sys.path.append('/ufs/guido/lib/python')
```

### dir()函数

内置函数`dir()`用于找出模块中定义了哪些名字。它返回一个排序后的字符串列表：

```python
>>> import fibo, sys
>>> dir(fibo)
['__name__', 'fib', 'fib2']
>>> dir(sys)
['__displayhook__', '__doc__', '__excepthook__', '__loader__', '__name__',
 '__package__', '__stderr__', '__stdin__', '__stdout__',
 '_clear_type_cache', '_current_frames', '_debugmallocstats', '_getframe',
 '_home', '_mercurial', '_xoptions', 'abiflags', 'api_version', 'argv',
 'base_exec_prefix', 'base_prefix', 'builtin_module_names', 'byteorder',
 'call_tracing', 'callstats', 'copyright', 'displayhook',
 'dont_write_bytecode', 'exc_info', 'excepthook', 'exec_prefix',
 'executable', 'exit', 'flags', 'float_info', 'float_repr_style',
 'getcheckinterval', 'getdefaultencoding', 'getdlopenflags',
 'getfilesystemencoding', 'getobjects', 'getprofile', 'getrecursionlimit',
 'getrefcount', 'getsizeof', 'getswitchinterval', 'gettotalrefcount',
 'gettrace', 'hash_info', 'hexversion', 'implementation', 'int_info',
 'intern', 'maxsize', 'maxunicode', 'meta_path', 'modules', 'path',
 'path_hooks', 'path_importer_cache', 'platform', 'prefix', 'ps1',
 'setcheckinterval', 'setdlopenflags', 'setprofile', 'setrecursionlimit',
 'setswitchinterval', 'settrace', 'stderr', 'stdin', 'stdout',
 'thread_info', 'version', 'version_info', 'warnoptions']
```

如果不带参数，dir()列出当前已定义的名称：

```python
>>> a = [1, 2, 3, 4, 5]
>>> import fibo
>>> fib = fibo.fib
>>> dir()
['__builtins__', '__name__', 'a', 'fib', 'fibo', 'sys']
```

注意，它列出了所有类型的名称： 变量、模块、函数等。

`dir()`不会列出内置的函数和变量的名称。如果想列出这些内容，它们定义在标准模块builtins中：

```python
>>> import builtins
>>> dir(builtins)
['ArithmeticError', 'AssertionError', 'AttributeError', 'BaseException',
 'BlockingIOError', 'BrokenPipeError', 'BufferError', 'BytesWarning',
 'ChildProcessError', 'ConnectionAbortedError', 'ConnectionError',
 'ConnectionRefusedError', 'ConnectionResetError', 'DeprecationWarning',
 'EOFError', 'Ellipsis', 'EnvironmentError', 'Exception', 'False',
 'FileExistsError', 'FileNotFoundError', 'FloatingPointError',
 'FutureWarning', 'GeneratorExit', 'IOError', 'ImportError',
 'ImportWarning', 'IndentationError', 'IndexError', 'InterruptedError',
 'IsADirectoryError', 'KeyError', 'KeyboardInterrupt', 'LookupError',
 'MemoryError', 'NameError', 'None', 'NotADirectoryError', 'NotImplemented',
 'NotImplementedError', 'OSError', 'OverflowError',
 'PendingDeprecationWarning', 'PermissionError', 'ProcessLookupError',
 'ReferenceError', 'ResourceWarning', 'RuntimeError', 'RuntimeWarning',
 'StopIteration', 'SyntaxError', 'SyntaxWarning', 'SystemError',
 'SystemExit', 'TabError', 'TimeoutError', 'True', 'TypeError',
 'UnboundLocalError', 'UnicodeDecodeError', 'UnicodeEncodeError',
 'UnicodeError', 'UnicodeTranslateError', 'UnicodeWarning', 'UserWarning',
 'ValueError', 'Warning', 'ZeroDivisionError', '_', '__build_class__',
 '__debug__', '__doc__', '__import__', '__name__', '__package__', 'abs',
 'all', 'any', 'ascii', 'bin', 'bool', 'bytearray', 'bytes', 'callable',
 'chr', 'classmethod', 'compile', 'complex', 'copyright', 'credits',
 'delattr', 'dict', 'dir', 'divmod', 'enumerate', 'eval', 'exec', 'exit',
 'filter', 'float', 'format', 'frozenset', 'getattr', 'globals', 'hasattr',
 'hash', 'help', 'hex', 'id', 'input', 'int', 'isinstance', 'issubclass',
 'iter', 'len', 'license', 'list', 'locals', 'map', 'max', 'memoryview',
 'min', 'next', 'object', 'oct', 'open', 'ord', 'pow', 'print', 'property',
 'quit', 'range', 'repr', 'reversed', 'round', 'set', 'setattr', 'slice',
 'sorted', 'staticmethod', 'str', 'sum', 'super', 'tuple', 'type', 'vars',
 'zip']
```

### 包（Package）

包是一种管理 Python 模块命名空间的方式，采用的是“点分模块名称”方式。例如，模块名`A.B`表示包A中一个名为B的子模块。

例如，一个包的结构可能是这样子的（分层的文件系统）：

```
sound/                          Top-level package
      __init__.py               Initialize the sound package
      formats/                  Subpackage for file format conversions
              __init__.py
              wavread.py
              wavwrite.py
              aiffread.py
              aiffwrite.py
              auread.py
              auwrite.py
              ...
      effects/                  Subpackage for sound effects
              __init__.py
              echo.py
              surround.py
              reverse.py
              ...
      filters/                  Subpackage for filters
              __init__.py
              equalizer.py
              vocoder.py
              karaoke.py
              ...
```

导入这个包时，Python搜索sys.path中的目录以寻找这个包的子目录。

为了让Python将目录当做包，目录下必须包含__init__.py文件。最简单的情况下，__init__.py可以只是一个空的文件，也可以为包执行初始化代码或设置__all__变量。

用户可以从包中导入单独的模块，例如：

```python
import sound.effects.echo
```

这样就加载了子模块sound.effects.echo。但是使用时必须用它完整名称来引用。

```python
sound.effects.echo.echofilter(input, output, delay=0.7, atten=4)
```

另一种导入子模块的方法是：

```python
from sound.effects import echo
```

这样也加载了子模块echo，而且使用它时可以不用包前缀访问：

```python
echo.echofilter(input, output, delay=0.7, atten=4)
```

还有一种导入方式就是直接导入函数或者变量：

```python
from sound.effects.echo import echofilter
```

这样可以直接访问函数：

```python
echofilter(input, output, delay=0.7, atten=4)
```

注意，使用`from package import item`时，item既可以是包的子模块（或子包），也可以是包中定义的一些其它的名称，比如函数、类或者变量。import语句首先测试item在包中是否有定义；如果没有，那么假定它是一个模块，并尝试加载它。如果仍没有找到，则抛出ImportError异常。

而使用`import item.subitem.subsubitem`这样的方式时，除了最后一项其它每项必须是一个包；最后一项可以是一个模块或一个包，但不能是前一个项目中定义的类、函数或变量。

### 从包中导入*

如果这样写`from sound.effects import *`，会发生什么呢？理想情况是到文件系统中寻找包里面含有哪些子模块，并把它们全部导入。这样导入子模块可能会产生一些意想不到的副作用，这些作用本应该只有当子模块是显式导入时才会发生。

唯一的解决办法是为包提供显式的索引。import语句约定：如果包中的__init__.py代码定义了一个名为__all__的列表，那么在`from package import *`时，就把这个列表中的所有模块名字导入。

例如，在文件`sound/effects/__init__.py`中包含下列代码：

```python
__all__ = ["echo", "surround", "reverse"]
```

这样`from sound.effects import *`只会导入这三个模块。

如果没有定义__all__，`from sound.effects import *`则不会从sound.effects包中导入所有的子模块到当前命名空间；它只保证sound.effects包已经被导入（可能会运行__init__.py中的任何初始化代码），然后导入包中定义的任何名称。这包括由__init__.py定义的任何名称（以及它显式加载的子模块）。还包括这个包中已经由前面的import语句显式加载的子模块。

### 包内引用

如果一个包是子包（比如上面的sound包），可以使用绝对导入来引用兄弟包的子模块。例如，如果模块sound.filters.vocoder需要使用sound.effects包中的echo模块，可以使用`from sound.effects import echo`。

还可以用`from module import name`形式的import语句进行相对导入。这些导入使用前导的点号表示相对导入的是从当前包还是上级的包。以surround模块为例：

```python
from . import echo
from .. import formats
from ..filters import equalizer
```

注意，相对导入基于当前模块的名称。因为主模块的名字总是`__main__`，Python应用程序的主模块必须总是用绝对导入。

### 包含多个目录的包

包还支持一个特殊的属性，__path__。该变量初始化一个包含__init__.py所在目录的列表。这个变量可以修改；这样做会影响未来包中包含的模块和子包的搜索。

虽然通常不需要此功能，它可以用于扩展包中的模块的集合。


---

参考资料：

[Python3 Documentation - Tutorial](https://docs.python.org/3/tutorial/modules.html)
