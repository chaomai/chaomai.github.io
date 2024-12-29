---
title: David Beazley - Python Concurrency From the Ground Up笔记
date: 2019-05-13 00:17:47
categories:
    - concurrency
tags:
    - python
    - concurrency
    - pycon
---

Python Concurrency From the Ground Up，来自[捕蛇者说](https://pythonhunter.org/episodes/1)的推荐，是David Beazley在PyCon 2015上的talk。在这个talk中，他边讲边写、外加开点玩笑，可以说David在各种意义上，都是并发的专家，很值得一看。视频和代码如下：

* [David Beazley - Python Concurrency From the Ground Up: LIVE! - PyCon 2015](https://www.youtube.com/watch?v=MCs5OvhV9S4)
* [dabeaz/concurrencylive](https://github.com/dabeaz/concurrencylive)

本文记录了这个talk的主要内容，并加上了我自己的理解。

这个talk从零实现了一个能支持多客户端并发访问的server，server计算了菲波那切数列第n项的值，为了展示blocking调用，用的是普通的递归实现`fib(n) = fib(n-1) + fib(n-2)`。同时还写了两个简单的client来测试server性能：`perf1.py`无限循环`fib(30)`，并输出每次调用的时间；`perf2.py`无限循环`fib(1)`，测试ops，这个调用是立即返回的。

# 版本1：简单单线程
```python
def fib_server(address):
    sock = socket(AF_INET, SOCK_STREAM)
    sock.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)
    sock.bind(address)
    sock.listen(5)
    while True:
        client, addr = sock.accept()
        print("Connection", addr)
        fib_handler(client)

def fib_handler(client):
    while True:
        req = client.recv(100)
        if not req:
            break
        n = int(req)
        result = fib(n)
        resp = str(result).encode('ascii') + b'\n'
        client.send(resp)
    print("Closed")
```

这个实现的问题是，server无法同时响应多个client。当某个client连接上后，`fib_handler(client)`会执行到这个client断开为止。

# 版本2：多线程
在版本1的基础上，

```python
def fib_server(address):
    # ...
        print("Connection", addr)
        Thread(target=fib_handler, args=(client,), daemon=True).start()
```

此时server可以响应多个client。但是由于GIL的存在，python是无法利用多个cpu核心的，因此，

1. `perf1.py`的结果（每次调用的时间）会随着`perf1.py`实例的增加而增加。同一时刻只能响应一个client，其他的等待，因此每次调用的时间大概是`单个client调用时间的均值 * perf1.py实例个数`。
2. `perf2.py`的结果（ops）会受其他调用的影响，ops会下降，`n`越大，下降越多。David这里提到了，GIL的一个特性是会把优先级给到计算更加密集的任务上，而os的调度却不会受这个影响，运行时间短的任务优先级更高。

python的每个线程实际上都有os实际的线程与其对应，用`ps -o cmd,nlwp <pid>`可看，但为何如此调度与其实现有关。对于os，Linux 2.6.23开始采用的是[Completely Fair Scheduler](https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt)；FreeBSD和macOS采用的是[Multilevel feedback queue](https://en.wikipedia.org/wiki/Scheduling_(computing))的调度算法，这也就解释了上述为什么运行时间较短的任务优先级更高。因为总能在规定的时间片内运行完成，不会被调度到后面的队列。

# 版本3：多线程+进程池
在版本2的基础上，

```python
from concurrent.futures import ProcessPoolExecutor as Pool

def fib_handler(client):
    # ...
        future = pool.submit(fib, n)
        result = future.result()
```

由于需要与子进程通信，需要序列化和反序列化数据，引入了额外的开销，因此`perf2.py`的ops会下降；但与此同时，server端处理计算任务是在单独的进程中，相当于计算任务的调度是由os来完成了，结合[版本2：多线程](#版本2：多线程)中对os调度的解释，基本不受其他计算更加密集的任务影响。

# 版本4：事件循环和协程
回看前三个版本的server，使用线程，本质上是为了解决blocking。而blocking主要发生在等待io的时候，可以考虑只有当io ready的时候才去处理socket。

```python
tasks = deque()
recv_wait = { }   # Mapping sockets -> tasks (generators)
send_wait = { }

def run():
    while any([tasks, recv_wait, send_wait]):
        while not tasks:
            # No active tasks to run
            # wait for I/O
            can_recv, can_send, _ = select(recv_wait, send_wait, [])
            for s in can_recv:
                tasks.append(recv_wait.pop(s))
            for s in can_send:
                tasks.append(send_wait.pop(s))

        task = tasks.popleft()
        try:
            why, what = next(task)   # Run to the yield
            if why == 'recv':
                # Must go wait somewhere
                recv_wait[what] = task
            elif why == 'send':
                send_wait[what] = task

            else:
                raise RuntimeError("ARG!")
        except StopIteration:
            print("task done")

class AsyncSocket(object):
    def __init__(self, sock):
        self.sock = sock
    def recv(self, maxsize):
        yield 'recv', self.sock
        return self.sock.recv(maxsize)
    def send(self, data):
        yield 'send', self.sock
        return self.sock.send(data)
    def accept(self):
        yield 'recv', self.sock
        client, addr = self.sock.accept()
        return AsyncSocket(client), addr
    def __getattr__(self, name):
        return getattr(self.sock, name)

def fib_server(address):
    sock = AsyncSocket(socket(AF_INET, SOCK_STREAM))
    sock.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)
    sock.bind(address)
    sock.listen(5)
    while True:
        client, addr = yield from sock.accept()  # blocking
        print("Connection", addr)
        tasks.append(fib_handler(client))

def fib_handler(client):
    while True:
        req = yield from client.recv(100)   # blocking
        if not req:
            break
        n = int(req)
        result = fib(n)
        resp = str(result).encode('ascii') + b'\n'
        yield from client.send(resp)    # blocking
    print("Closed")

tasks.append(fib_server(('',25000)))
run()
```

这个版本的核心在`def run()`，利用`yield`实现了协程。当遇到io时，`yield`跳出当前执行，`select`判断io ready后，才去读写socket。要注意的是，这个版本虽然没有使用多线程，但server是可以服务多个client的，因为在某个client的socket没有ready的时候，server可以做其他的事情。不过由于是单线程，对于所有client提交的计算任务，server只能逐一执行。协程并不能帮助解决多线程中GIL的问题，因为并没有利用到多个cpu核心，[版本2：多线程](#版本2：多线程)的两个性能问题，这里都存在。

我觉得这里的`yield`和事件循环是用的很出彩，换做是我，我首先考虑到的是`select`出ready的socket，然后进行读写。弊端在于需要把业务逻辑套在一个大循环里面，每次都先调用`select`，在不同的socket ready的时候，使用相应的业务逻辑进行处理。

# 版本5：事件循环和协程+多进程
在版本4的基础上，

```python
def fib_handler(client):
    # ...
        future = pool.submit(fib, n)
        result = future.result()    #  Blocks
```

考虑到上一个版本无法利用多个cpu核心进行计算，那么如果像[版本3：多线程+进程池](#版本3：多线程+进程池)一样把`fib`放入`pool`中，是否能解决问题呢？放入以后，会发现当某个协程执行到'future.result()'的时候就会阻塞，直到`pool`中的任务计算完毕，相当于server主线程会逐一等待每个计算任务。[版本2：多线程](#版本2：多线程)的两个问题，仍然存在。

# 版本6：事件循环和协程+多进程
在版本5的基础上，

```python
future_wait = { }

future_notify, future_event = socketpair()

def future_done(future):
    tasks.append(future_wait.pop(future))
    future_notify.send(b'x')

def future_monitor():
    while True:
        yield 'recv', future_event
        future_event.recv(100)

tasks.append(future_monitor())

def run():
    # ...
        task = tasks.popleft()
        try:
            why, what = next(task)   # Run to the yield
            if why == 'recv':
                # Must go wait somewhere
                recv_wait[what] = task
            elif why == 'send':
                send_wait[what] = task
            elif why == 'future':
                future_wait[what] = task
                what.add_done_callback(future_done)

# ...

def fib_handler(client):
    # ...
        future = pool.submit(fib, n)
        yield 'future', future
        result = future.result()    #  Blocks
```

最后这个实现首先将`result = fib(n)`，放入了`pool`并得到一个`future`，`yeild`之后为这个future添加计算完成后的回调`future_done`。这个`pool`可以是线程池（无法利用多个cpu），也可以是进程池。比较hacking的地方是用`socketpair`把计算ready转变了socket ready。

# 总结
除去实现中用到的一些技巧，这个talk把GIL和blocking的影响、要用什么样的方式来绕开这些问题，以及事件循环和协程讲的很明白。

P.S. 小插曲，现场写的时候，David把

```python
can_recv, can_send, _ = select(recv_wait, send_wait, [])
```

写成了

```python
can_recv, can_send, [] = select(recv_wait, send_wait, [])
```

最后有个提问者表示，*the empty listing was just some incredible next-level thing that I was just not capable of*。

至于为什么unpack到空list，原因如下，

```python
# 首先unpack到一个变量的list是可以的
>>> a, b, [x, y] = 1, 2, [10, 20]  # a=1, b=2, x=10, y=20

# 如果对一个空list进行unpack，由于没有东西可以unpack，所以可以解到另一个空的list里面
>>> a, b, [] = 1, 2, []   # a=1, b=2
```

