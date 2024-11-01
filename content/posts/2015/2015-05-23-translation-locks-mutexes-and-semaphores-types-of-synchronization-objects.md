title: "译 - Locks, Mutexes, and Semaphores: Types of Synchronization Objects"
date: 2015-05-23 22:12:13
description: C++ Concurrency in Action作者的文章，介绍了一些关于并发的基础概念：锁、互斥量和信号量。
categories:
    - concurrency
tags:
    - synchronize
    - reading
    - cpp
    - cpp11
---

# Locks

lock是一个抽象的概念。一个基本的前提就是一个lock保护着某种共享资源的访问。如果你own一个lock，那么你就能访问被保护的共享资源。如果你没有own lock，那么你就不能够访问这个共享资源。

为了own一个lock，你首先需要某种lockable对象。然后你从那个对象获得lock。这个操作精确的术语可能会有很多。例如，如果你有一个lockable对象XYZ，你可以：

* acquire the lock on XYZ,
* take the lock on XYZ,
* lock XYZ,
* take ownership of XYZ,
* or some similar term specific to the type of XYZ

lock的概念也意味着某种exclusion：有时，你可能不能获得ownership of a lock，接着将要执行的操作将会fail、或block。就前者而言，操作将会返回某些错误码或异常，以指明take ownership的操作尝试失败。而后者，只有当这个操作take ownership，它才会返回，而这需要系统里的其他线程完成一些工作才能使得这个发生。

exclusion最常见的形式是一个简单的计数：lockable对象有最大数目的owners。如果达到了这个数目，那么接下来任何尝试获取a lock on it都不会成功。因此，这需要我们有某种机制（当我们完成操作的时候，放弃ownership）。这通常叫做unlocking，但是同样的术语可能不同。例如，你可以：

* release the lock on XYZ,
* drop the lock on XYZ,
* unlock XYZ,
* relinquish ownership of XYZ,
* or some similar term specific to the type of XYZ

当你以合适的方式relinquish ownership，如果所需的条件都满足了，那么一个被block的尝试获得锁的操作现在将会继续。

例如一个lockable对象只允许有3个owners，那么第4个尝试获得lock的操作将block。当3个中的某个owner 释放了lock，那么第4个尝试获得lock的操作将会成功。

# Ownership

"own" a lock的意思视lockable对象确切的类型而定。某些lockable对象会对ownership有非常严格的定义：this specific thread owns the lock, through the use of that specific object, within this particular scope.

在其他情况下，这个定义会更不稳定，ownership of the lock会更抽象。在这些情况下，ownership can be relinquished by a different thread or object than the thread or object that acquired the lock.

注：其实就想看这篇文章里说Ownership的部分，后面的就不翻译了:)

# Reference

* 原文：[Locks, Mutexes, and Semaphores: Types of Synchronization Objects](https://www.justsoftwaresolutions.co.uk/threading/locks-mutexes-semaphores.html)
