---
title: python符号表、module和package
date: 2019-03-05 22:12:40
categories:
    - python
tags:
    - symtable
---

# symbol table
symbol table（符号表）是用于编译器或解释器中的一个数据结构，存储了源码中每个符号以及关联的信息。不同的作用域可能会有各自的符号表。在静态语言中，符号表尤为重要。

## python variable
python中的变量与c/c++的不一样，是绑定到对象的symbolic name。`aa`，`bb`都是绑定到一个list对象的名字。

```python
aa = [1, 2, 3]
bb = aa
aa[0] = 666
```

## python symbol table
虽然python是动态语言，没有编译时类型检查，但python也有符号表。python的符号表通过编译器由AST生成，用于计算每个标识符的作用域，最终符号表和AST会被共同用于生成字节码。`symtable`模块提供了关于标识符的作用域等信息，还能够输出在这些作用域中引用到的变量是哪个。

```python
# add.py
def outer(aa):
    def inner():
        bb = 1
        return aa + bb + cc
    return inner

src = open('add.py', 'r').read()
import symtable
table = symtable.symtable(src, 'src.py', 'exec')

def describe_symbol(sym):
    assert type(sym) == symtable.Symbol
    print("Symbol:", sym.get_name())
    for prop in [
            'referenced', 'imported', 'parameter',
            'global', 'declared_global', 'local',
            'free', 'assigned', 'namespace']:
        if getattr(sym, 'is_' + prop)():
            print('    is', prop)

for i in table.get_children()[0].get_children()[0].get_symbols():
    describe_symbol(i)

Symbol: bb
    is referenced
    is local
    is assigned
Symbol: aa
    is referenced
    is free
Symbol: cc
    is referenced
    is global
```

free variable：如果变量在一个代码块内使用（在代码内被绑定），但是并没有在其中定义，那么这个变量是自由变量。顺便提一句，使用了自由变量的函数就是闭包（closure）。
global variable：如果变量代码在模块级别被绑定，这个变量是全局变量。
local variable：如果变量在代码块内被绑定，这个变量是局部变量。

# python module
module（模块）是一个包含python定义和语句的文件，文件名由module名和`.py`后缀构成。module名可由全局变量`__name__`获取。这个文件可被作为脚本使用或用于repl中，module中的定义可以被其他module或mainmodule导入。

例如：`p.py`，
```python
def add(a, b):
    return a + b
```

在repl中，
```python
# 把module名p加入当前的符号表
import p
p.add(1, 2)

Symbol: p
    is imported
    is local

# 把p中的add导入当前符号表
from p import add
add(1, 2)

Symbol: add
    is imported
    is local

# 把p中的除_开头的所有名字导入当前符号表
from p import *
```

一个module，可以包含可执行语句和函数定义，这些语句是用于初始化module的，仅仅在第一次导入这个模块的时候执行，作为脚本运行的时候也会执行。module作为脚本执行的时候，`__name__`会被设置为`"__main__"`，很常见的一个语句是`if __name__ == "__main__":`，作为脚本执行的时候，运行`if`内的语句。

每个module有自己私有的符号表。module内部的全局变量不会与外部的产生冲突，另一个角度看，如果知道模块内的全局变量名称，可以通过`modname.itemname`的方式访问到。通过`dir()`函数可以得到module内定义的名字。

导入module时，module名字的search path为，
1. 内置的module
2. `sys.path`
    这个变量的值由下列几项初始化得到，
    * 脚本当前目录
    * `PYTHONPATH`环境变量
    * the installation-dependent default

    `sys.path`初始化以后，可以在程序中修改。

## python package
package是一种通过`modnameA.submodnameB`来组织module的方式。python会把包含`__init__.py`文件的文件夹当做package。`__init__.py`文件可以直接留空，也可以加入初始化package的代码或设置`__all__`变量（用于控制`from package imrpot *`的时候，导入哪些）。

导入的方式为，
* `from package import item`。import语句首先测试item是否在package中定义，如果没有，那么假设item是一个module并尝试load。最终如果没有找到，那么会抛出`ImportError`异常。
* `import item.subitem.subsubitem`，最后一个item可以是module或package，但不能是类、函数或变量，其他必须是package。

# class和module
在使用的时候，class和module有一些相似的地方，但是二者完全不同。

* class是创建带有属性和方法的实例的蓝图，支持继承等。这些module都不能做。
* module仅仅只是组织代码的方式。

# References
1. [Python internals: Symbol tables, part 1](https://eli.thegreenplace.net/2010/09/18/python-internals-symbol-tables-part-1)
2. [symtable — Access to the compiler’s symbol tables¶](https://docs.python.org/2/library/symtable.html)
