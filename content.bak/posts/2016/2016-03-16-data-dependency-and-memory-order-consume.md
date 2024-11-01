title: Data Dependency and memory_order_consume
date: 2016-03-16 19:08:06
categories:
    - concurrency
tags:
    - 'data dependency'
---

# Data Dependency

`memory_order_consume`是关于data dependency的，我的理解是更细粒度的acquire-release，通过使用`memory_order_consume`，可以避免对其他无依赖的数据强加同步。

关于data dependency，有两个关系（两个关系都有传递性），

* carries-a-dependency-to：在单线程中，如果操作A的结果被用作操作B的操作符，那么操作A carries-a-dependency-to 操作B。
* dependency-ordered-before：在线程之间，有标记为`memory_order_release`，`memory_order_acq_rel`或`memory_order_seq_cst`的store操作A，如果标记为`memory_order_consume`的load操作B read了被store的数据，则操作A dependency-ordered-before 操作B。（操作A和B都是原子的）

线程之间，如果A dependency-ordered-before B，那么A happens-before B。

# 使用场景

对于这种memory ordering，一个典型的应用就是load一个指针指向的数据，其中load是原子操作。

```cpp
struct X {
  int i;
  std::string s;
};

std::atomic<X*> p;
std::atomic<int> a;

void create_x() {
  X* x = new X;
  x->i = 42;
  x->s = "hello";
  a.store(99, std::memory_order_relaxed);
  p.store(x, std::memory_order_release);
}

void use_x() {
  X* x;
  while (!(x = p.load(std::memory_order_consume))) {
    std::this_thread::sleep_for(std::chrono::microseconds(1));
  }

  assert(x->i == 42);
  assert(x->s == "hello");
  assert(a.load(std::memory_order_relaxed) == 99);
}
```

上述代码中，`p`的store是`memory_order_release`的，它的load是`memory_order_consume`的，循环保证了load能够读取到store的指针，因此当load能够读取到store的指针时，`p`的store dependency-ordered-before 它的load。所以`p`的store happens-before 它的load。

又因为`x`的store happens-before `p`的store，`p`的load happens-before `x`的load。

因此`x`的store happens-before `x`的load，对于`x`的两个assert不会触发，但`a`的load读取到的值是没有保证的。

# kill_dependency

用`kill_dependency`可以显式地打破依赖链。

对于如下代码（取自[stackoverflow](http://stackoverflow.com/questions/7150395/what-does-stdkill-dependency-do-and-why-would-i-want-to-use-it)），

```cpp
r1 = x.load(memory_order_consume);
r2 = r1->index;
do_something_with(a[std::kill_dependency(r2)]);
```

使用`kill_dependency`让编译器知道不需要再次读取`r2`的值，因此编译器可以将代码优化为，

```cpp
predicted_r2 = x->index; // unordered load
r1 = x; // ordered load
r2 = r1->index; // ordered load
do_something_with(a[predicted_r2]); // 不需要再次读取r2
```

甚至优化为，

```cpp
predicted_r2 = x->index; // unordered load
predicted_a  = a[predicted_r2]; // get the CPU loading it early on
r1 = x; // ordered load
r2 = r1->index; // ordered load
do_something_with(predicted_a);
```
