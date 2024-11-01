title: Fence和非原子操作的ordering
date: 2016-03-20 20:04:21
categories:
    - concurrency
tags:
    - fence
---

# Fence

除了在原子操作中标记memory ordering外，还可以单独使用fence指定memory ordering。Fence是全局的操作，它影响所执行线程中其他原子操作的ordering。

```cpp
atomic<bool> x,y;
atomic<int> z;

void write_x_then_y() {
  x.store(true,memory_order_relaxed);
  atomic_thread_fence(memory_order_release);
  y.store(true,memory_order_relaxed);
}

void read_y_then_x() {
  while(!y.load(memory_order_relaxed));
  atomic_thread_fence(memory_order_acquire);
  if(x.load(memory_order_relaxed))
    ++z;
}
```

上面的代码中，如果没有显式的fence，`z`的值是不确定的。

关于fence，有几个synchronizes-with规则：

* 如果acquire操作能读取到位于release fence后面store的写入的值，那么这个fence synchronizes-with acquire操作。
* 如果位于acquire fence前面的load操作能够读取到release操作的值，那么这个release操作synchronizes-with acquire fence。
* 如果位于acquire fence前面的load操作能够读取到位于release fence后面的store写入的值，那么release fence synchronizes-with acquire fence。

对于上面的代码，因为y的load能够读取到前面写入的值（由于fence存在，保证了ordering），所以release fence synchronizes-with acquire fence。

# Ordering Nonatomic

```cpp
bool x;
atomic<bool> y;
atomic<int> z;

void write_x_then_y() {
  x = true;
  atomic_thread_fence(memory_order_release);
  y.store(true,memory_order_relaxed);
}

void read_y_then_x() {
  while(!y.load(memory_order_relaxed));
  atomic_thread_fence(memory_order_acquire);
  if(x)
    ++z;
}
```

现在把前面例子中的x换为普通的x，z的值仍然是有保证的，**y必须是原子的**。fence保证了x的store和y的store，以及y的load和x的load之间的ordering，而y的store和load之间有happens-before关系，因此x的store和load之间也有happens-before关系（传递）。
