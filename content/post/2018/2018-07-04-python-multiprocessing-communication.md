---
title: Python进程间通信总结
date: 2018-07-04 22:09:50
categories:
    - python
tags:
    - concurrency
---

# Python进程间通信总结

Python的multiprocessing支持使用类似threading模块的API来创建进程，multiprocessing提供了本地和远程并发，有效的避免了线程中的GIL。下面对Python进程间通信的方法做了简单的总结，并列举了相应的例子。

# 消息传递

## pipe

```python
def worker(conn):
    i = conn.recv()
    print(i)
    conn.close()

if __name__ == '__main__':
    parent_conn, child_conn = Pipe()
    p = Process(target=f, args=(child_conn,))
    p.start()
    parent_conn.send(1)
    p.join()
```

## queue

`multiprocessing.Queue`使用pepe和locks/semaphores实现了进程间的共享队列。任何picklable对象都可以通过`queue`传递。

```python
def worker(q):
    i = q.get()
    print(i)

if __name__ == '__main__':
    queue = multiprocessing.Queue()
    p = multiprocessing.Process(target=worker, args=(queue,))
    p.start()

    queue.put(1)
    queue.close()
    queue.join_thread()
    p.join()
```

# 同步原语

## Barrier

一种简单的同步原语，用于固定数目的进程相互等待。当所有进程都调用`wait`以后，所有进程会同时开始执行。

```python
def worker(b):
    i = b.wait()
    print(i)
    if i == 0:
        print('passed the barrier')

if __name__ == '__main__':
    barrier = multiprocessing.Barrier(2)
    p = multiprocessing.Process(target=worker, args=(barrier,))
    p.start()

    q = multiprocessing.Process(target=worker, args=(barrier,))
    q.start()

    p.join()
    p.join()
```

## Semaphore

## BoundedSemaphore

## Condition

条件变量。多个进程可以等待同一个条件变量，直到他们被另一个进程通知。条件变量默认使用`RLock`，可以提供自己的锁，但必须是`Lock`或者`RLock`对象。

```python
def producer(q, c):
    c.acquire()
    while len(q) == 0:
        c.wait()

    i = q.pop()
    print(i)
    c.release()

def consumer(q, c):
    for i in range(10):
        c.acquire()
        q.append(i)
        c.notify_all()
        c.release()
        time.sleep(1)

if __name__ == '__main__':
    queue = []
    cond = multiprocessing.Condition()
    p1 = multiprocessing.Process(target=consumer, args=(queue, cond,))
    p1.start()

    p2 = multiprocessing.Process(target=consumer, args=(queue, cond,))
    p2.start()

    q = multiprocessing.Process(target=producer, args=(queue, cond,))
    q.start()

    p1.join()
    p2.join()
    q.join()
```

## Event

`Event`提供了一种简单的方法来实现进程间的状态传递。event类似一个flag，状态是set或者unset。

```python
def wait_for_event(e):
    print('wait_for_event: starting')
    e.wait()
    print('wait_for_event: e.is_set()->', e.is_set())

def wait_for_event_timeout(e, t):
    print('wait_for_event_timeout: starting')
    e.wait(t)
    print('wait_for_event_timeout: e.is_set()->', e.is_set())

if __name__ == '__main__':
    e = multiprocessing.Event()
    w1 = multiprocessing.Process(name='block', target=wait_for_event, args=(e,))
    w1.start()

    w2 = multiprocessing.Process(name='nonblock', target=wait_for_event_timeout, args=(e, 2))
    w2.start()

    print('main: waiting before calling Event.set()')
    time.sleep(3)
    e.set()
    print('main: event is set')
```

## Lock

# 状态共享

## shared memory map

通过使用`Value`或`Array`，数据可以被存储在shared memory map中。

```python
def f(n, a):
    n.value = 3.1415927
    for i in range(len(a)):
        a[i] = -a[i]

if __name__ == '__main__':
    num = multiprocessing.Value('d', 0.0)
    arr = multiprocessing.Array('i', range(10))

    p = multiprocessing.Process(target=f, args=(num, arr))
    p.start()
    p.join()

    print(num.value)
    print(arr[:])
```

`Value`和`Array`在shared memory分配了ctypes对象。默认情况下，返回的值是使用同步方法封装过的对象，即线程安全的。其中，参数`d`和`i`代表了值的类型。

# References

1. [Passing Messages to Processes](https://pymotw.com/3/multiprocessing/communication.html)
2. [multiprocessing — Process-based parallelism](https://docs.python.org/3/library/multiprocessing.html)
