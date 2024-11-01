title: Release Sequences
date: 2016-03-17 19:10:31
categories:
    - concurrency
tags:
    - 'release sequence'
---

# Release Sequences

如果

* 有标记为`memory_order_release`，`memory_order_acq_rel`或`memory_order_seq_cst`的store，
* 和标记为`memory_order_consume`，`memory_order_acquire`或`memory_order_seq_cst`的load，
* 并且在**操作链中的每个操作都load上一个操作write的值**，

那么这个操作链构成一个**release sequence**，并且
* 初始的store synchronizes-with 用`memory_order_acquire`或`memory_order_seq_cst`标记的最后的load；
* 或初始的store dependency-ordered-before 用`memory_order_consume`标记的最后的load。

操作链中的RMW操作可以被标记为任意一种memory ordering。

```cpp
vector<int> queue_data;
atomic<int> count;

void f1() {
  queue_data.push_back(1); // (1)
  count.store(number_of_items, memory_order_release); // (2)
}

void f2() {
  while(true) {
    int item_index;
    if((item_index = count.fetch_sub(1, memory_order_acquire)) <= 0) { // (3)
      continue;
    }
    process(queue_data[item_index-1]);
  }
}

thread a(f1);
thread b(f2);
thread c(f2);
```

在上述代码中，执行`f2`的有两个线程，由于acquire-release语义，(2)肯定是与第一个(3)有synchronizes-with关系的。后面还有一个(3)，第一个(3)写入的值被第二个(3)读取，因此操作链构成一个release sequence，由因为最后一个(3)是`memory_order_acquire`（无论是哪个线程中的`fetch_sub`作为最后一个），因此有(2)synchronizes-with最后一个(3)。
