title: "译 - Understand `std::atomic::compare_exchange_weak()` in C++11"
date: 2015-06-09 00:00:49
description: 原文是stackoverflow上的一个关于compare_exchange_weak()问题和相应的答案，我做了简单的翻译和整理。
categories:
    - concurrency
tags:
    - atomic
    - reading
    - cpp
    - cpp11
    - cas
---

<blockquote class="blockquote-center">

原文是stackoverflow上的[一个关于`compare_exchange_weak()`问题和相应的答案](http://stackoverflow.com/questions/25199838/understanding-stdatomiccompare-exchange-weak-in-c11)。

</blockquote>

## Question

```cpp
bool compare_exchange_weak (T& expected, T val, ..);
```

`compare_exchange_weak()`是C++11中提供的compare-exchange原语之一。之所以是**weak**，是因为即使在对象的值等于`expected`的情况下，也返回false。这是因为在某些平台上的**spurious failure**，这些平台使用了一系列的指令（而不是像在x86上一样，使用单条的指令）来实现CAS。在这种平台上，context switch, reloading of the same address (or cache line) by another thread等，将会导致这条原语失败。由于不是因为对象的值（不等于`expected`）导致的操作失败，因此是`spurious`。相反的，it's kind of timing issues。

但是困扰我的是C++11标准（ISO/IEC 14882）里的，

> 29.6.5 .. A consequence of spurious failure is that nearly all uses of weak compare-and-exchange will be in a loop.

为什么in **nearly all uses**都必须在一个loop中？这是不是意味着因为有spurious failures，当它失败的时候，我们将会loop？如果这是原因，那么为什么我们还要那么麻烦的使用`compare_exchange_weak()`，并且自己写loop？我们可以直接使用`compare_exchange_strong()`，我认为这样可以让我们摆脱spurious failures。`compare_exchange_weak()`通常在什么情况下使用？

另一个相关的问题是，在Anthony的"C++ Concurrency In Action"一书中写到，

```cpp
//Because compare_exchange_weak() can fail spuriously, it must typically
//be used in a loop:

bool expected=false;
extern atomic<bool> b; // set somewhere else
while(!b.compare_exchange_weak(expected,true) && !expected);

//In this case, you keep looping as long as expected is still false,
//indicating that the compare_exchange_weak() call failed spuriously.
```

为什么在loop的条件中有!expected？它是不是防止了所有线程可能会starve，并且在一段时间里make no progress？

**Edit：(最后一个问题)**

在那些没有单条硬件CAS指令的平台上，weak和strong版本都是使用LL/SC来实现的（像ARM，PowerPC等）。那么下面的两种loop之间是否有区别？如果有的话，为什么？（对我来说，我觉得它们应该有相似的性能）。

```cpp
// use LL/SC (or CAS on x86) and ignore/loop on spurious failures
while (!compare_exchange_weak(..))
{ .. }

// use LL/SC (or CAS on x86) and ignore/loop on spurious failures
while (!compare_exchange_strong(..))
{ .. }
```

你们都提到，在loop在可能会存在性能上的区别。这也在C++11（ISO/IEC 14882）标准中提到，

> When a compare-and-exchange is in a loop, the weak version will yield better performance on some platforms.

但是根据上面的分析，两个版本的loop应该有相同或类似的性能。我遗漏了什么吗？

## Answer 1

### Why doing exchange in a loop?

通常来说，你想要在进行下一步的工作前完成当前的工作，因此，你把`compare_exchange_weak`放入了一个loop，以便一直尝试exchange直到成功（也就是说，返回`true`）。

要注意的是，`compare_exchange_strong`也常常用在loop中。它不会因为spurious failure而失败，但会因为concurrent writes而失败。

### Why to use `weak` instead of `strong`?

很简单：spurious failure并不会经常发生，因此这不会有大的性能损失。相反，tolerate这种failure允许在某些平台上有更高效的`weak`版本的实现（与`strong`相比较）。`strong`必须每次都检查spurious failure，并处理它，这是有代价的。

因此，`weak`在某些平台上比`strong`更快使得`weak`被使用。

### When should you use `weak` and when `strong`?

这个[参考文献](http://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange)指明了什么时候用`weak`，什么时候用`strong`：

> When a compare-and-exchange is in a loop, the weak version will yield better performance on some platforms. When a weak compare-and-exchange would require a loop and a strong one would not, the strong one is preferable.

> 当compare-and-exchange在loop中，weak版本在某些平台上会有更高的性能。当一个weak的compare-and-exchange需要一个loop，而strong的不需要，那么用strong的更合适。

因此这个答案很容易记住：如果你使用loop仅仅是由于spurious failure，那么不要使用loop；用`strong`。如果你无法避免的要使用loop，那么就是用`weak`。

### Why is `!expected` in the example?

这视实际情况和期望的语义而定，但是通常it is not needed for correctness。忽略它会产生一个非常相似的语义。仅仅在另一个线程可能会把值reset为`false`的情况下，这个语义才会有细微的差别（但是我没有找到一个比较meaningful的例子），Tony D的评论有更详细的解释。

**Tony D：**

"Why is !expected in the example? It is not needed for correctness. Omitting it would yield the same semantics."

并非如此。假设第一个exchange由于b已经是`true`而失败，此时`expected`变为`true`。如果没有`&& !expected`的话，它只会再次loop，并且（傻傻的）将`true`与`true`交换。这可能会成功并且退出`while` loop。But could exhibit meaningfully different behavior if b had meanwhile changed back to false, in which case the loop would continue and may ultimately set b true yet again before breaking.

当其他线程写入`true`时，这是个简单快捷的方法：我们只需要终止，而不是再次尝试写入`true`。

### About your last question

> 但是根据上面的分析，两个版本的loop应该有相同或类似的性能。我遗漏了什么吗？

在[Wikipedia](http://en.wikipedia.org/wiki/Load-link/store-conditional)：

> Real implementations of LL/SC do not always succeed if there are no concurrent updates to the memory location in question. Any exceptional events between the two operations, such as a context switch, another load-link, or even (on many platforms) another load or store operation, will cause the store-conditional to spuriously fail. Older implementations will fail if there are any updates broadcast over the memory bus.

因此，LL/SC在发生context switch的时候将会fail spuriously。现在，`strong`版本的会使用它“自己的小loop”来检测spurious failure，并且通过重试来处理它。要注意的是，它自己的loop要比一个普通的CAS loop要复杂，因为它必须区分spurious failure（并处理它）和由于并发访问导致的失败（这会导致返回值为`false`）。`weak`版的并没有这样自己的loop。

既然你在两个例子中都显式的使用了loop，在`strong`版本中是完全没有必要使用loop的。结果是，在`strong`版本中，做了两次failure的检查；一次是`compare_exchange_strong`做的（由于它要区分spurious failure和并发访问，这要更复杂），另一是你自己的loop做的。这样代价高昂的检查时没有必要的，`weak`快的原因就在这里。

Also note that your argument (LL/SC) is just one possibility to implement this. There are more platforms that have even different instruction sets. In addition (and more importantly), note that std::atomic must support all operations for all possible data types, 因此即使你声明了一个一千万字节的结构体，你还是可以用compare_exchange来做这个。即使当CPU有CAS指令，你也不用能够CAS一千万字节，因此编译器会生成其他的指令（可能是acquire lock，然后进行non-atomic compare and swap，接着release lock）。现在，想想当swap一千万字节的时候会发生什么。因此尽管spurious error对于8 byte的exchanges可能是非常罕见的，在这种情况下，它可能更常见。

## Answer 2

**To be continued...**

## References

1. [Understanding std::atomic::compare_exchange_weak() in C++11](http://stackoverflow.com/questions/25199838/understanding-stdatomiccompare-exchange-weak-in-c11)
