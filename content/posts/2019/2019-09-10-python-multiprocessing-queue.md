---
title: python multiprocessing.Queue模块源码阅读
date: 2019-09-10 12:35:20
categories:
    - python
tags:
    - concurrency
---

之前有记录过python [Queue](/2018/2018-07-04-python-multiprocessing-communication)的使用，以及[multiprocessing.Process模块](/2019/2019-09-09-python-multiprocessing-process)。现在看看`multiprocessing.Queue`的具体工作方式（本文基于Python 3.7.4）。

`multiprocessing.Queue`定义在queues.py中，除此之外还定义了`SimpleQueue`和`JoinableQueue`，是FIFO队列。

类似`multiprocessing.Process`，首先导入了`context`，判断当前系统类型，并相应地使用对应的实现。使用的`multiprocessing.Queue()`的时候，实际上是调用了`context`中的`Queue()`方法，设置了`ctx`。
```python
def Queue(self, maxsize=0):
    '''Returns a queue object'''
    from .queues import Queue
    return Queue(maxsize, ctx=self.get_context())
```

# SimpleQueue
`SimpleQueue`很简单，其实就是一个带锁的pipe，
* 主进程和子进程分别使用各自的lock，实现写入pipe和读取pipe是并发安全的
    之所以`put()`和`get()`可以使用不同的lock，是因为pipe两端的读写已经是并发安全的了。
* 用`multiprocessing.Pipe`来实现消息传递，支持多读多写

> 关于`os.pipe`和`multiprocessing.Pipe`
> * `os.pipe`：在Linux上底层访问的是传统的POSIX pipes，单向，使用encode/decode序列化
> * `multiprocessing.Pipe`：使用`multiprocessing.Connection`实现，在Linux上使用POSIX sockets完成数据发送，双向，使用pickle/unpickle序列化

# Queue
`Queue`可以看做在`SimpleQueue`的基础上，增加了生产者端的发送buffer、支持设置队列大小，以及`get()`和`put()`的无阻塞调用。

![python queues](images/2019/python%20queues.jpg)

## 初始化
`__init__()`初始化了这么几个比较重要的变量，
* `_maxsize`：队列最大size
* `_reader`，`_writer`：`multiprocessing.Connection`实例，负责数据的收发
* `_rlock`：`multiprocessing.Lock`，进程间共享，保证`_reader.recv_bytes()`并发安全
* `_wlock`：`multiprocessing.Lock`，进程间共享，保证`_writer.send_bytes()`并发安全
* `_sem`：队列长度信号量，计数器初始化为`_maxsize`
* `_notempty`：条件变量，同步生产者放入`_buffer`和`_buffer`中数据的发送
* `_buffer`：每个进程有自己独立的buffer，线程安全，size其实是由`_sem`来控制
* `_thread`：生产者数据发送线程

## put
首先`_sem.acquire()`，计数器-1，如果不为零，说明队列还可以写入。
```python
if not self._sem.acquire(block, timeout):
    raise Full
```

然后获得`_notempty`的锁，如果发送线程未创建，则创建。追加元素至buffer后，`_notempty.notify()`通知发送线程。生产者中数据的发送由单独的线程完成，主线程只负责将数据放入buffer。
```python
with self._notempty: # acquire保护_notempty和_thread的修改
    if self._thread is None:
        self._start_thread()
    self._buffer.append(obj)
    self._notempty.notify()
```

## feed线程
`_start_thread()`创建thread对象后，设置`daemon`属性为`True`，目的是随主线程的退出而退出，不必手动添加停止的逻辑。接着就启动线程。

具体发送时，不断等待buffer中有元素；如果buffer中有元素，`popleft()`并发送，直到发送完。

为何`_notempty`的锁在`popleft()`就释放了？
1. buffer是`collections.deque`，本身就是并发安全的
2. `_notempty`锁的目的其实是为了保护`_notempty`的修改，和`put()`中的目的类似，但并不是为了保护buffer
```python
while 1:
    try:
        nacquire()
        try:
            if not buffer:
                nwait()
        finally:
            nrelease()
        try:
            while 1:
                obj = bpopleft()
                if obj is sentinel:
                    debug('feeder thread got sentinel -- exiting')
                    close()
                    return

                # serialize the data before acquiring the lock
                obj = _ForkingPickler.dumps(obj)
                if wacquire is None:
                    send_bytes(obj)
                else:
                    wacquire()
                    try:
                        send_bytes(obj)
                    finally:
                        wrelease()
        except IndexError:
            pass
    except Exception as e:
        # ......
```

## get
**对于`block=True`的情况**
类似`SimpleQueue`，一直等待`_reader.recv_bytes()`，直到收到数据。在`_reader.recv_bytes()`以后，`_sem.release()`，计数器+1，表示从队列消耗了一个元素。此时如果有进程调用`_sem.acquire()`并在等待，那么`_sem.release()`会唤醒其中一个等待的进程。

**对于`block=False`的情况**
由于不能阻塞，因此不能一直等待`_reader.recv_bytes()`。

1. 如果在`timeout`内`_rlock`都没有获得锁，则返回`Empty`
2. `_reader.poll()`（底层还是使用`select`来实现的）判断是否在`timeout`内`_reader`可读，如果不可读，则返回`Empty`
3. 最后`_reader.recv_bytes()`，并`_sem.release()`

```python
def get(self, block=True, timeout=None):
    if block and timeout is None:
        with self._rlock:
            res = self._recv_bytes()
        self._sem.release()
    else:
        if block:
            deadline = time.monotonic() + timeout
        if not self._rlock.acquire(block, timeout):
            raise Empty
        try:
            if block:
                timeout = deadline - time.monotonic()
                if not self._poll(timeout): # 此时的timeout已经减去了等待_rlock.acquire的时间
                    raise Empty
            elif not self._poll():
                raise Empty
            res = self._recv_bytes()
            self._sem.release()
        finally:
            self._rlock.release()
    # unserialize the data after having released the lock
    return _ForkingPickler.loads(res)
```

## 序列化和反序列化
序列化和反序列化使用的是pickle来实现，那么如何判断消息的边界？`Connection`中定义了一个规则，在`header`存放长度。
```python
def _send_bytes(self, buf):
    n = len(buf)
    # For wire compatibility with 3.2 and lower
    header = struct.pack("!i", n)
    if n > 16384:
        # The payload is large so Nagle's algorithm won't be triggered
        # and we'd better avoid the cost of concatenation.
        self._send(header)
        self._send(buf)
    else:
        # Issue #20540: concatenate before sending, to avoid delays due
        # to Nagle's algorithm on a TCP socket.
        # Also note we want to avoid sending a 0-length buffer separately,
        # to avoid "broken pipe" errors if the other end closed the pipe.
        self._send(header + buf)

def _recv_bytes(self, maxsize=None):
    buf = self._recv(4)
    size, = struct.unpack("!i", buf.getvalue())
    if maxsize is not None and size > maxsize:
        return None
    return self._recv(size)
```

## close和join_thread
由于生产者启动了一个线程来负责发送，元素首先append到buffer，然后发送。在进程结束时，如何确保元素发送完毕的问题？

在启动feed线程的时候，创建了两个`Finalize`对象，`_finalize_close`和`_finalize_join`，前者的优先级较高，并set了`_close`和`_jointhread`。当进程退出的时候，会自动地先后调用这两个函数（基于`atexit`实现），
1. `_finalize_close`：会append元素`_sentinel = object()`到buffer，feed线程如果看到`_sentinel`，会调用`close()`并停止服务
2. `_finalize_join`：join feed线程

```python
if not self._joincancelled:
    self._jointhread = Finalize(
        self._thread, Queue._finalize_join,
        [weakref.ref(self._thread)],
        exitpriority=-5
        )

# Send sentinel to the thread queue object when garbage collected
self._close = Finalize(
    self, Queue._finalize_close,
    [self._buffer, self._notempty],
    exitpriority=10
    )
```

模块同样也提供了`close()`和`join_thread()`方法，调用的其实就是`_close`和`_jointhread`。如果先手动调用，然后`aexit`调用，那不会回有问题？首次调用后，都会从`util`模块的`_finalizer_registry`中移除，因此不会存在重复调用`_finalize_close`和`_finalize_join`的问题。

## deadlock
这几个stackoverflow（[1](https://stackoverflow.com/questions/31665328/python-3-multiprocessing-queue-deadlock-when-calling-join-before-the-queue-is-em)，[2](https://stackoverflow.com/questions/31708646/process-join-and-queue-dont-work-with-large-numbers)，[3](https://stackoverflow.com/questions/26738648/script-using-multiprocessing-module-does-not-terminate)）的问题很类似，在官方的文档中也有提及（[python 2](https://docs.python.org/2.7/library/multiprocessing.html#pipes-and-queues)，[python 3](https://docs.python.org/3/library/multiprocessing.html#all-start-methods)），总结下来就是，*join一个调用put的进程，且这个进程尚未把buffer中所有的元素写入pipe时*，可能会导致死锁。

```python
import os
import multiprocessing

def work(q):
    q.put('1'*10000000)

if __name__ == "__main__":
    print(os.getpid())
    q = multiprocessing.Queue()

    p = multiprocessing.Process(target=work, args=(q,))

    p.start()
    p.join()

    print(q.get())
```

原因是，`Queue`使用了os的pipe来进行数据的传输，而pipe的大小是有限的。如果数据过大，`_writer.send_bytes()`在写入数据到pipe的时候会阻塞。如果此时join这个子进程，那进程本身已经卡住了，join永远等不到进程结束。

# JoinableQueue
`JoinableQueue`基于`Queue`实现，覆盖了`put()`，并新增了，
* `task_done()`：表示先前放入队列中的元素被取走了，由消费者调用
* `join()`：阻塞直到队列中所有元素都被取走

## 初始化
`__init__()`调用了`Queue`的初始化函数，并额外初始化了，
* `_unfinished_tasks`：信号量，表示队列中当前未取走的元素
* `_cond`：条件变量，同步`join()`和`task_done()`

## put
和`Queue`的`put()`基本一致。多出来的一点是，每次`put()`都会对`_unfinished_tasks`的计数器+1。
```python
    def put(self, obj, block=True, timeout=None):
        # ...
        with self._notempty, self._cond:
            if self._thread is None:
                self._start_thread()
            self._buffer.append(obj)
            self._unfinished_tasks.release()
            self._notempty.notify()
```

## task_done和join
`task_done()`对`_unfinished_tasks`的计数器进行-1，如果计数器为0，则通知等待在`join()`的进程。

```python
def task_done(self):
    with self._cond:
        if not self._unfinished_tasks.acquire(False):
            raise ValueError('task_done() called too many times')
        if self._unfinished_tasks._semlock._is_zero():
            self._cond.notify_all()

def join(self):
    with self._cond:
        if not self._unfinished_tasks._semlock._is_zero():
            self._cond.wait()
```

# Reference
1. [Python os.pipe vs multiprocessing.Pipe](https://stackoverflow.com/questions/15720120/python-os-pipe-vs-multiprocessing-pipe)
2. [Daemon is not daemon, but what is it?](https://laike9m.com/blog/daemon-is-not-daemon-but-what-is-it,97/)
3. [Python 中的 multiprocess.Queue](https://blog.dreamfever.me/2019/04/21/python-zhong-de-multiprocess-queue/)
4. [Python 3 Multiprocessing queue deadlock when calling join before the queue is empty](https://stackoverflow.com/questions/31665328/python-3-multiprocessing-queue-deadlock-when-calling-join-before-the-queue-is-em)
5. [Process.join() and queue don't work with large numbers](https://stackoverflow.com/questions/31708646/process-join-and-queue-dont-work-with-large-numbers)
6. [Script using multiprocessing module does not terminate](https://stackoverflow.com/questions/26738648/script-using-multiprocessing-module-does-not-terminate)
7. [https://docs.python.org/3/library/multiprocessing.html#all-start-methods](https://docs.python.org/3/library/multiprocessing.html#all-start-methods)
8. [Joining multiprocessing queue takes a long time](https://stackoverflow.com/questions/42350933/joining-multiprocessing-queue-takes-a-long-time)
