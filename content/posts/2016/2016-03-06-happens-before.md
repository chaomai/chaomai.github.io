---
title: Happens-before
date: 2016-03-06 21:51:54
categories:
    - concurrency
tags:
    - happens-before
---

取自[preshing博客](http://preshing.com/)上的几篇文章（[1](http://preshing.com/20120913/acquire-and-release-semantics/)，[2](http://preshing.com/20130702/the-happens-before-relation/)，[3](http://preshing.com/20130823/the-synchronizes-with-relation/)）。除了部分翻译外，还有自己的理解。

# Happens-before关系

假设A和B两个操作是由多线程程序执行的，如果**A happens-before B**，那么A对内存的操作在**B被执行前**对执行B的线程切实可见。

关于happens-before要注意的是一下看起来自相矛盾的两点。因为happens-before所描述的是操作之间的关系，这个关系是**独立于时间的**，并不是happening before。

## Happens-before并不意味着happening before

```cpp
int A = 0;
int B = 0;
void foo() {
  A = B + 1;              // (1)
  B = 1;                  // (2)
}
```

以上代码中，只看program order的话，(1)是happens-before(2)的。但编译器可能会对上面的代码进行reorder（用clang++ 3.7 -O2没有发生），使得B的store**先于A完成**。

从happens-before定义来看，(1)对内存的修改必须在(2)执行前切实可见，也就是说A的store必须影响到B的store。但从这个例子来看，A的store并未影响到A，就算没有(1)，(2)的行为也是一样的，这就等价于(1)的操作是[可见的](http://preshing.com/20130702/the-happens-before-relation/)。

因此(1)和(2)行为并不违背happens-before，happens-before并不意味着happening before。

## Happening before并不意味着happens-before

假设下面对的int的store和load都是原子的，有两个线程分别执行两个函数。就program order而言，(1)和(2)，(3)和(4)之间有happens-before关系。再假设在运行时，(2)在(3)之前完成，(3)读到了1。

但是并不意味着(2)和(3)之间有happens-before关系。

```cpp
int isReady = 0;
int answer = 0;
void publishMessage() {
  answer = 42;                      // (1)
  isReady = 1;                      // (2)
}
void consumeMessage() {
  if (isReady)                      // (3) <-- Let's suppose this line reads 1
    printf("%d\n", answer);         // (4)
}
```

happens-before关系仅仅在标准指明的地方有。C++11中并未规定在普通的store和load之间有happens-before关系。进一步看，(1)和(4)之间也没有。因此(1)和(4)是可以被编译器或CPU reordered的。即使(3)读到了1，(4)可能打印0。


# 单线程中的happens-before关系

如果操作A和B是由同一个线程执行的，且就program order而言，A的语句位于B之前，那么A **happens-before** B。

然而这并不是唯一实现happens-before关系的方法。

# 多线程中的happens-before关系

上面提到了单线程中的happens-before关系是如何产生的，下面来看多线程中的happens-before关系，C++11指出可以通过acquire和release语义，在不同线程的操作中实现happens-before。

## Acquire和Release语义

Acquire语义：Acquire语义是一个属性，这个属性只能应用于从共享内存中的**read操作**，无论这些read是RMW还是普通的load。Acquire语义保证了在program order上位于read-acquire之后的read和write不会被编译器和CPU reordered到read-acquire之前。
Release语义：Release语义也是一个属性，这个属性只能应用于从共享内存中的**write操作**，无论这些read是RMW还是普通的store。Release语义保证了在program order上位于write-release之前的read和write不会被编译器和CPU reordered到write-release之后。

Raymond Chen的另一个解释，
一个带有acquire语义的操作不允许后续的内存操作提前到该操作之前执行，相对的，一个带有release语义的操作不允许前面的内存操作被滞后到该操作之后执行。

关于Acquire和Release语义，[这里](http://hedengcheng.com/?p=725)还有一个比较好的解释。

### 通过显式的CPU fence指令实现acquire和release语义

下面代码中有两个全局变量`A`和`Ready`，两个线程分别执行两段代码，`Ready`作为flag表示`A`的write是否完成。

```cpp
int A = 0;
int Ready = 0;

// thread 1
A = 42;
#StoreStore
Ready = 1;

// thread 2
int r1 = Ready;
#LoadLoad
int r2 = A;
```

通过两个fence，可以保证当线程2发现`r1 == 1`时，`A`的值是1，进而保证`r2 == 1`。

规范的来说就是，对`Ready`的write synchronizes-with对`Ready`的read。

## Synchronizes-with关系

Synchronizes-with用于描述源码级操作的内存影响（describe ways in which the memory effects of source-level operations），即使是非原子操作，也能够保证结果是对其他线程可见。一个较为常见的事情是，无论何时在两个线程间有**synchronizes-with**关系（一般是在不同的线程间）那么在这些操作之间都会有**happens-before**关系。

### 一个Write-Release能够Synchronize-with一个Read-Acquire的

C++11标准规定了，一个对**原子对象M**执行**release**操作的**原子操作A** *synchronize-with*一个对**M**执行**acquire**操作的**原子操作B**，且能够得到以A为起始的release sequence操作的所有副作用。

```cpp
struct Message {
  clock_t     tick;
  const char* str;
  void*       param;
};
Message g_payload;
std::atomic<int> g_guard(0);

void SendTestMessage(void* param) {
  // Copy to shared memory using non-atomic stores.
  g_payload.tick  = clock();
  g_payload.str   = "TestMessage";
  g_payload.param = param;
  // Perform an atomic write-release to indicate that the message is ready.
  g_guard.store(1, std::memory_order_release);
}

bool TryReceiveMessage(Message& result) {
  // Perform an atomic read-acquire to check whether the message is ready.
  int ready = g_guard.load(std::memory_order_acquire);
  if (ready != 0) {
    // Yes. Copy from shared memory using non-atomic loads.
    result.tick  = g_payload.tick;
    result.str   = g_payload.str;
    result.param = g_payload.param;

    return true;
  }
  // No.
  return false;
}
```

上述代码中，为了能够把`g_payload`安全的在线程中传递，使用了acquire和release。当`TryReceiveMessage`中`g_guard`的read-acquire执行完后，**如果`ready`是1**，那么`g_payload`的三个成员一定被成功写入。

与前面的标准对照，可以看到，

* 原子操作A是`SendTestMessage`中的write-release；
* 原子对象M是`g_guard`；
* 原子操作B是`TryReceiveMessage`中的read-acquire。

前面所说的takes its value from any side effect in the release sequence headed by A，这里指read-acquire能够读取到write-release所写的值。**如果读取到了**，那么synchronize-with关系就出现了。此时，两个线程间就有了happens-before关系。有时这也叫做synchronize-with或happens-before“边”。

![](/images/2016/happens_before_two-cones.png)

标准中还保证了只要有synchronize-with边存在，happens-before关系就能够**扩展到临近的操作**。对应到上例中就是，当其他线程读取`g_payload`时，保证能够读取到对`g_payload`写入的值。

### 运行时关系

想要通过静态的分析代码，来寻找代码中的synchronize-with关系是错误的。synchronize-with是**运行时关系**。

![](/images/2016/happens_before_no-cones.png)

如果`g_guard`读取的过早，线程1还没有写入`g_guard`，那么就没有synchronize-with关系。

### 其他实现Synchronize-with的方法

![](/images/2016/happens_before_org-chart.png)
