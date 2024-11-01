title: "C++ Concurrency in Action (1) - Hello, world of concurrency in C++!"
date: 2015-05-17 19:35:23
description: Hello, world of concurrency in C++!的笔记。
categories:
    - concurrency
tags:
    - reading
    - cpp
    - cpp11
    - 'c++ concurrency in action'
---

# 关于此系列文章

最近在看这本书，这个系列文章是我在看书过程中的笔记，记录了一些我觉得关键的地方和自己的思考。如果能帮助到你，I will be very happy :).

# 关于C++ Concurrency in Action

{% blockquote 并发编程网 http://ifeve.com/c-plus-plus-concurrency-in-action/ 《C++ Concurrency in Action》中文版 %}
本书是一本基于C++11新标准的并发和多线程编程深度指南。从std::thread、std::mutex、std::future和std::async等基础类的使用，到内存模型和原子操作、基于锁和无锁数据结构的构建，再扩展到并行算法、线程管理，最后还介绍了多线程代码的测试工作。本书的附录部分还对C++11新语言特性中与多线程相关的项目进行了简要的介绍，并提供了C++11线程库的完整参考。
{% endblockquote %}

2014年初要出的中文版，只是到现在还没有，看英文吧。

# Hello, world of concurrency in C++

在C++中实现多线程，可以写出行为有保证的可移植的代码。

### Appraoches to concurrency

#### 多进程

消息传递由进程间通信实现，但是，

* 由于操作系统的有很多保护机制来避免一个进程难以修改另一个的数据，因此实现通讯的方式复杂或者慢；
* 有固有的开销，启动进程需要时间（系统需要分配资源等）。

#### 多线程

操作系统要做的更少，灵活的共享内存是有代价的。

* 内存一致性

## Why use concurrency

* separation of concerns
* performance

### Ways to use concurrency

* task parallelism, data parallelism；
* 使用现有的并行计算能力来解决更大的问题；
* （两种方式有着不同的关注点：一个是利用并行来缩短任务的时间；另一个是在任务处理时间一定的情况下，并行的运行多个任务来加大处理量。）

### When not to use concurrency

* 实现并发的cost>收益；
* 线程的启动需要时间来给os分配相关的内核资源和栈空间，如果线程完成的时间很短，那可能启动的时间就占据了运行时间的大部分；
* 由于系统的资源有限，线程是一种有限的资源,线程越多，os必须进行更多的context switching。
