---
title: python multiprocessing.Process模块源码阅读
date: 2019-09-09 15:16:20
categories:
    - python
tags:
    - concurrency
---

之前有记录过[python进程间通信](/2018/2018-07-04-python-multiprocessing-communication)的几个方式，现在来看看这个模块的具体的是怎样工作的（本文基于Python 3.7.4）。

`multiprocessing.Process`典型的使用方式为，
```python
import multiprocessing

def f(name):
    print('hello', name)

p = multiprocessing.Process(target=f, args=('world',))
p.start()
p.join()
```

以这段代码为例，看看进程的执行过程。

![python process](/images/2019/python%20process.jpg)

## import
`import multiprocessing`，导入multiprocessing的时候，做了一些初始化的工作。
1. multiprocessing的包导入了`context`，这个模块主要是判断当前系统类型，并相应地使用对应的实现。在`sys.platform != 'win32'`的系统上，例如：mac和linux，默认使用`fork`来创建子进程。
2. `context`模块中，
    * 导入了`process`
    * 定义了`Process`类，这个类继承自`process.BaseProcess`，覆盖了`process.BaseProcess`的`_Popen`方法，目的是根据系统类型，在初始化的时候根据系统类型，选择相应的`Popen`实现
3. `process`模块中，
    * 导入了`util`，注册了`_exit_function`，被注册的函数会在解释器正常终止时执行（子进程主动调用，主进程自动调用）
    * 此时在父进程里，初始化了几个`process`模块的全局变量，
        * `_current_process`：代表当前进程
        * `_process_counter`：进程数计数器
        * `_children`：子进程的集合

## 创建`Process`实例
`p = multiprocessing.Process(target=f, args=('world',))`，实际上是调用`process.BaseProcess`的`__init__()`方法进行初始化，并使用了`process`模块的全局变量初始化实例变量，以及获取当前进程的pid，
* `count = next(_process_counter)`
* `self._identity = _current_process._identity + (count,)`
* `self._config = _current_process._config.copy()`
* `self._parent_pid = os.getpid()`

## 启动
`p.start()`创建子进程，首先做了几个检查，
1. 如果已经创建了子进程，那么不能再次创建
2. 只能start由当前进程创建的`Process`实例
3. 不允许创建daemon进程的子进程

接着`_children`中清理当前已经结束的进程，然后调用`self._Popen(self)`开始创建子进程。使用`os.fork()`创建子进程，用法和posix的fork一样，不多说。要注意的一点是，调用`os.pipe()`创建了`parent_r`和`child_w`，而`parent_r`将会被用于`join()`的实现。

子进程中，执行`_bootstrap()`进行初始化和运行`target`，
1. 初始化了`process`模块的全局变量，`_current_process`、`_process_counter`、`_children`
2. 清空`util._finalizer_registry`，执行`util._run_after_forkers()`
3. `run()`运行`target`
4. `util._exit_function()`做进程结束后的收尾工作
    * `util._exit_function()`会调用`_run_finalizers()`，这个函数会把优先级高于`minpriority`的`finalizer`执行一遍
        ```python
        def _run_finalizers(minpriority=None):
            '''
            Run all finalizers whose exit priority is not None and at least minpriority

            Finalizers with highest priority are called first; finalizers with
            the same priority will be called in reverse order of creation.
            '''
        ```
    * 默认情况下子进程的`_finalizer_registry`是空的，没有任何的`finalizer`会被执行，但可以通过`multiprocessing.util.Finalize`手动的进行注册，来完成一些收尾工作，例如：关闭db连接。
        ```python
        class Finalize(object):
            '''
            Class which supports object finalization using weakrefs
            '''
            def __init__(self, obj, callback, args=(), kwargs=None, exitpriority=None):
                # ......
                _finalizer_registry[self._key] = self
        ```

## join
创建子进程的时候，也创建了pipe `parent_r`，并set `self.sentinel = parent_r`，且关闭了`child_w`。此时唯一打开`child_w`的进程是子进程，唯一打开`parent_r`的是主进程。主进程调用`join()`时，实际上是等待`parent_r`变为可读状态（`wait`返回）。
```python
from multiprocessing.connection import wait
if not wait([self.sentinel], timeout):
    return None
```

那么何时`wait`返回？`wait`循环调用`select`，当`parent_r` ready的时候，wait返回，
* `parent_r`可读
* 所有writing side被关闭

当子进程结束的时候，os会关闭这个进程关联的所有fd，又因为主进程已经关闭了`child_w`，所以此时`parent_r` ready，主进程中的`join()`也就返回了。

# Reference
1. [multiprocessing — Process-based parallelism](https://docs.python.org/3/library/multiprocessing.html)
2. [Why should you close a pipe in linux?](https://stackoverflow.com/questions/19265191/why-should-you-close-a-pipe-in-linux)
