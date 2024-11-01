title: Memory Ordering
date: 2016-03-13 22:34:28
categories:
    - concurrency
tags:
    - 'memory ordering'
---

Memory ordering描述了CPU访问系统内存，执行load和store的顺序。Memory ordering包括编译时编译器生成的和运行时CPU生成的。为了高效地执行指令，**只要不影响单线程程序的行为**，编译器和CPU常常会对指令进行memory reordering，使得访问内存的操作不会按照程序代码中指定的顺序执行。

在单线程程序中，可以忽略reordering的存在；在多线程程序中，mutex，semaphores等互斥方法会保证在相关函数的调用周围没有reordering。在多核环境下（或对称多处理器架构）下，用C、C++等编写lock-free代码时，memory reordering是可观察到的，是重点要考虑的问题。

## Compiler Reordering

从源代码得到CPU指令的过程中，编译器会做很多事情，其中之一就是reordering。（例子摘自[Preshing on Programming](http://preshing.com/20120625/memory-ordering-at-compile-time/)）

```cpp
int Value;
int IsPublished = 0;
void sendValue(int x) {
  Value = x;
  IsPublished = 1;
}
```

如果`IsPublished = 1;`被reordered到`Value = x;`之前，那么`IsPublished`作为flag的作用就丧失了。既是是在单线程中，也会有问题。如果该线程在两次store之间被抢占，那么其他使用`Value`的线程就会访问到`Value`的旧值，而不是新值。

### Compiler Barriers

防止编译器reordering最简单的办法莫过于使用compiler barriers，在不想被reordered的两个操作（load和store）之间加入compiler Barrier`asm volatile("" ::: "memory")`，但在多核环境下，这并不足够。还需要下面的CPU fence来提供运行时memory barrier的保障。如果使用了CPU fence，那么fence也会作为compiler Barrier。

另一种compiler barrier是函数调用，**无论函数是否包含**compiler barrier，除了inline函数、声明带有`pure`[属性](http://lwn.net/Articles/285332/)的函数和使用了链接时代码生成的函数，大多数函数调用都可以作为compiler barrier。因为编译器根本不知道该函数调用是否会修改先前的值，也不知道这个值是否在函数调用返回后会被继续使用，如果进行了reordering，那么很可能会违反一开始提到的原则。而对于包含的函数，无论是不是inline的，都可以作为compiler barrier。

C/C++中，`volatie`会阻止编译器的优化，编译器[不会](http://hedengcheng.com/?p=725)对`volatie`变量间的操作进行reordering，**但是**`volatie`对处理器的reordering是无能为力的，并没有happens-before语义。与此同时，`volatie`也不能阻止多个线程的[并发访问](https://www.kernel.org/doc/Documentation/volatile-considered-harmful.txt)。对于下面的代码，无论`shared_data`是不是`volatie`，`spin_lock`都是必须的。

```c
spin_lock(&the_lock);
do_something_on(&shared_data);
do_something_else_with(&shared_data);
spin_unlock(&the_lock);
```

代码里的`spin_lock`也下面将说的memory barrier。

## Processor Reordering

除了编译器会进行reordering，CPU同样会。CPU的reordering仅在多核或多处理器环境下才是可见的。（例子摘自[Preshing on Programming](http://preshing.com/20120515/memory-reordering-caught-in-the-act/)）

考虑下面的一段代码，`thread1Func`和`thread2Func`分别在两个线程中运行，最后`r1`和`r2`的结果会是什么？为了阻止编译器的reordering，已经在store和load之间加入了compiler barrier。

```cpp
int X, Y;
int r1, r2;
void *thread1Func(void *param) {
  X = 1;
  asm volatile("" ::: "memory")；
  r2 = Y;
  return NULL;
};
void *thread2Func(void *param) {
  Y = 1;
  asm volatile("" ::: "memory")；
  r2 = X;
  return NULL;
};
```

由于是并发执行，两个线程的load和store会[交替执行](http://www.yebangyu.org/blog/2016/01/09/memoryconsistencyandcachecoherence/)，因此`r1`和`r2`的结果可能为，

```
r1=0 r2=1
r1=1 r2=0
r1=1 r2=1
```

但`r1=0 r2=0`也是完全可能的，如果重复的进行测试，那么这个结果会[频繁的发生](https://gist.github.com/ChaoMai/f756356369fe0b1d4859)。Intel在64 and IA-32 Architectures Software Developer's Manual中的8.2.3.4指出，

> At each processor, the load and the store are to different locations and hence may be reordered.

代码中每个线程的store和load是不同的内存位置，所以发生StoreLoad Reordering是完全可能的。

### Memory Barrier

上述例子的reordering只是众多memory reordering中的一种，主要有[四种memory reordering](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)，

* LoadLoad
* StoreStore
* LoadStore
* StoreLoad

类似compiler reordering，需要compiler barrier来阻止CPU的reordering，这里是memory barriers，也叫fence指令。Fence保证了fence之前的load或store和之后的load或store是严格有序的。针对以上四种reordering，有对应的四种barrier，`#LoadLoad`、`#StoreStore`、`#LoadStore`和`#StoreLoad`。对于现实中CPU的fence指令的行为，通常是上述[几种的融合](http://g.oswego.edu/dl/jmm/cookbook.html)，且还会有其他的效果。

```
Store1; #StoreLoad; Load2;
```

需要特别说明的是`#StoreLoad`，这个barrier是**唯一**能够保证上述例子不出现`r1=0 r2=0`的。`#StoreLoad`保证了在barrier前执行`Store1`对其他处理器是**可见的**，在barrier后执行的`Load2`能够得到**在barrier之后最新的值**（不一定是`Store1`的值）。`#StoreLoad`[防止](http://g.oswego.edu/dl/jmm/cookbook.html)了后续的load错误的使用`Store1`的值，而不是其他处理器在同一内存位置store的更新的值。`#StoreLoad`几乎在所有现代的多处理器上都是必须的，同时也是代价最高昂的一种barrier。

除了fence指令，memory barrier还有，

* C++11中，很多原子类型的操作；
* pthread中，mutex，spin_lock，semaphore的操作。
