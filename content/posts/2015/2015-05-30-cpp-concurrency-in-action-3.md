title: "C++ Concurrency in Action (3) - Sharing data between threads"
date: 2015-05-30 22:37:23
description: Sharing data between threads的笔记。
categories:
    - concurrency
tags:
    - mutex
    - reading
    - cpp
    - cpp11
    - 'c++ concurrency in action'
---

# C++ Concurrency in Action 3

## Problems with sharing data between threads

在同一进程中，线程间的数据共享，不仅仅是便捷，还可能造成问题。

线程间共享数据的问题，全都可以归结到修改数据的顺序。

* 如果所有数据都是只读的，那么不会有任何问题。
* 如果一个或多个线程修改它们之间共享的数据，现在的问题可能就会出现。

**Invariant**（不变条件）：关于某个特定的数据结构，始终为真的与君。

但是在多个线程修改共享数据时，invariant会被broken。

### Race conditions

并发中最常见的导致问题的原因。出现在两个或多个线程的执行结果依赖于相对的执行顺序。在某些情况下，race condition是无害的，但是如果race condition使得broken invariant，这将会导致问题。C++标准定义了一种特别的race condition - data race（由于并发的修改同一个对象），data race将会导致undefined behavior。

problematic race condition常常出现在完成一个操作需要修改两处或多处不同的数据。在一个线程正在修改数据，另一个可能在未修改完的时候就访问数据。

由于race condition是timing sensitive的，因此在debug的时候，debugger会影响程序的timing，问题就不会再现。

### Avoiding problematic race conditions

* 用某种保护机制来包装数据结构。

> 只有实际进行修改的线程才能看到invariants are broken的中间状态。

* 修改数据结构和invariants的设计，使得修改在一系列隐形的changes下完成，每个change可以保证invariants。

> 这一般指的就是lock-free。

* 将对数据结构的update作为transaction。

> 这就是software transactional memory（STM）。


## Protecting shared data with mutexes

mutex（mutual exclusion）：synchronization primitive。线程库保证一旦某个线程已经lock了某个mutex，其他所有尝试lock同一个mutex的线程，都必须等到那个成功lock这个mutex的线程unlock它。

这可以保证每个线程看到的都是self-consistent view的共享数据，避免broken invariant。

但mutex**不是**silver bullet！

### Using mutexes in C++

通过构造`std::mutex`的实例来创建一个mutex。

不推荐直接调用`std::mutex`的成员函数，尤其是`unlock()`。因为你必须在每段代码执行的尾部，包括异常中，都要记得unlock。`std::lock_guard`以**RAII**的方式来对mutex提供支持，在构造的时候lock，析构的时候unlock。

用类来封装。

### Structuring code for protecting shared data

如果类的成员函数返回了指向受保护数据的指针或引用，那岂不就是开了后门？

那么是不是禁止返回指向受保护数据的指针或引用就ok了？

还有一个没有考虑到的就是，不要向那些不在你控制下的被调用函数，传入指向受保护数据的指针或引用。

> Don't  pass  pointers  and  references  to  protected  data  outside  the  scope  of  the  lock,  whether  by returning them from a function, storing them in externally visible memory, or passing them as arguments to user-supplied functions.

### Spotting race conditions inherent in interfaces

考虑一个双向链表，如果要使得一个删除操作是线程安全的，那么要必须保证避免并发的访问三个结点（要删除的和两边的）。但这并不足以避免race condition，到目前为止，只能采用避免并发的访问整个list。

看下面的例子：

```cpp
template<typename T,typename Container=std::deque<T> >
class stack {
public:
	...
    bool empty() const;
    size_t size() const;
    T& top();
    T const& top() const;
    void push(T const&);
    void push(T&&);
    void pop();
    void swap(stack&&);
};
```

就算让`top()`返回copy，而不是reference，这个接口还是有race conditions。这并不是基于mutex的实现独有的问题，基于lock-free的实现也会有，这是接口设计的问题。

上例中，`empty()`和`top()`是不可靠的，因为在调用`empty()`或`size()`的线程使用它们返回的值之前，stack可能已经被其他线程改变了（可能已经empty），这时再去`top()`，stack已经不是调用`empty()`或`size()`时候的stack了。而在stack内使用mutex仅仅只能保证同一时刻，只有一个线程运行stack的成员函数，而这并不能解决这样的问题。

还有一个类似的问题是，`top()`和`pop()`之间，stack也有可能被修改。

```cpp
stack<int> s;
if(!s.empty()) {
    int const value=s.top();
    s.pop();
    do_something(value);
}
```

或许你觉得可以合并`top()`和`pop()`，但是`std::stack`这样分开设计不是没有理由的，简单来说，如果分开了，在return的时候，系统资源不足，copy构造函数失败，造成已经pop了，但是元素丢失（因为没有return成功）。

但是，比较头疼的是，要消除race condition就是要避免这样的分开设计（lock的粒度太小，mutex不能保护整个操作），进而避免上例中，不同线程的interleave。

**解法1：PASS IN A REFERENCE**

缺陷：

* 不实用，构造一个实例需要额外的时间或资源。
* 不总是可用，构造函数所需要的参数不是时时可以获得的。
* 需要类型可赋值，但是赋值不总是可用的，尤其是自定义的类型。

**解法2：REQUIRE A NO-THROW COPY CONSTRUCTOR OR MOVE CONSTRUCTOR**

There's only an exception safety problem with a value-returning pop() if the return by value can throw an exception.

缺陷：

* 限制只让那些有不抛出异常的构造函数或move构造函数的的类型使用stack，虽然可行，但是不通用。

**解法3：RETURN A POINTER TO THE POPPED ITEM**

优势：

* pointer可以被自由的copy，而不抛出异常。

缺陷：

* 普通的方式需要管理分配给对象的内存，内存管理的开销可能还大于return by value。

如果要使用，`std::shared_ptr`值得考虑，可以避免很多`new`和`delete`。

**解法 4：PROVIDE BOTH OPTION 1 AND EITHER OPTION 2 OR 3**

万金油，糅合1，2或1，3，要用什么，给用户自己选择。

**EXAMPLE DEFINITION OF A THREAD-SAFE STACK**

* 在接口中没有race condition，并糅合了1、3；
* 不可赋值，可copy（假设元素可copy）；
* stack为空时，`pop()`抛出异常，但是stack仍然可以工作；
* 简化的操作，使得数据得到了更好的控制，可以保证在每个操作中，mutex都是locked；
* 对`std::stack`进行了包装；
* 可以加入copy的支持，只要copy的时候lock mutex即可。

```cpp
#include <exception>
#include <memory>
#include <mutex>
#include <stack>
struct empty_stack: std::exception {
    const char* what() const throw();
};
template<typename T>
class threadsafe_stack {
private:
    std::stack<T> data;
    mutable std::mutex m;
public:
    threadsafe_stack(){}
    threadsafe_stack(const threadsafe_stack& other) {
        std::lock_guard<std::mutex> lock(other.m);
        data=other.data;
    }
    threadsafe_stack& operator=(const threadsafe_stack&) = delete;
    void push(T new_value) {
        std::lock_guard<std::mutex> lock(m);
        data.push(new_value);
    }
    std::shared_ptr<T> pop() {
        std::lock_guard<std::mutex> lock(m);
        if(data.empty()) throw empty_stack();
        std::shared_ptr<T> const res(std::make_shared<T>(data.top()));
        data.pop();
        return res;
    }
    void pop(T& value) {
        std::lock_guard<std::mutex> lock(m);
        if(data.empty()) throw empty_stack();
        value=data.top();
        data.pop();
    }
    bool empty() const {
        std::lock_guard<std::mutex> lock(m);
        return data.empty();
    }
};
```

**为什么m是mutable？**

`empty()`后跟const，说明`empty()`不能修改类的成员，除非成员被设置为mutable。

**Granularity**

细粒度（fine-grained）锁会导致保护不完全，锁的粒度太大会导致并发失去意义。

有时细粒度锁意味着需要多个mutex，而这可能会导致deadlock。

### Deadlock：the problem and a solution

如果有两个mutex，那么总是以同样的顺序lock mutex就不会死锁 了。但是有时以同样的顺序lock mutex并不能满足需求。

`std::lock`：可以一次性锁住两个或多个mutex，而不会deadlock。但是如果分别获得lock，它并不能保证不死锁。

```cpp
class some_big_object;
void swap(some_big_object& lhs,some_big_object& rhs);
class X {
private:
    some_big_object some_detail;
    std::mutex m;
public:
    X(some_big_object const& sd):some_detail(sd){}
    friend void swap(X& lhs, X& rhs) {
        if(&lhs==&rhs)
            return;
        std::lock(lhs.m,rhs.m);
        std::lock_guard<std::mutex> lock_a(lhs.m,std::adopt_lock);
        std::lock_guard<std::mutex> lock_b(rhs.m,std::adopt_lock);
        swap(lhs.some_detail,rhs.some_detail);
    }
};
```

上例中有这么几个要注意的地方：

**为什么要检查是否是不同的实例？**

`std::lock`做了这么一个事情：

> Locks the given Lockable objects lock1, lock2, ..., lockn using a deadlock avoidance algorithm to avoid deadlock. The objects are locked by an unspecified series of calls to lock, try_lock, unlock. If a call to lock or unlock results in an exception, unlock is called for any locked objects before rethrowing.

换句话说，作为参数的几个mutex都会被an unspecified series of calls to lock, try_lock, unlock一遍。

要注意的是，必须是[Lockable objects](http://en.cppreference.com/w/cpp/concept/Lockable)。

对于已经在外部lock的lockables，deadlock是无法保证不发生的。

如果某个mutex是locked，deadlock是不能够保证会避免的。

**`std::adopt_lock`是什么鬼？**

constexpr（常量表达式），tag type used to specify locking strategy 。其中std::adopt_lock：assume the calling thread already has ownership of the mutex。也就是说告诉`std::lock_guard`，给你的mutex已经锁上了，你就不要再lock一次了。

下面是`std::lock_guard`的构造函数的定义：

```cpp
explicit lock_guard(mutex_type& __m) : _M_device(__m)
{ _M_device.lock(); }

lock_guard(mutex_type& __m, adopt_lock_t) : _M_device(__m)
{ } // calling thread owns mutex
```

可以很明确的看出使用了`adopt_lock_t`后，`std::lock_guard`并没有去lock。

**会不会抛出异常？什么地方会？抛出会怎样？**

比较可能会，如果抛出，函数就退出了，不会进行swap；
`std::lock`可能会（确切的说是，`std::lock`内部`lock lhs.m`或`rhs.m`的时候可能会，接着异常会传出`std::lock`），如果抛出，函数就退出了，不会进行swap；
构造`lock_guard`不会，标准规定的。

`std::lock`不万能，如果分别获得lock，它并不能保证不deadlock。

### Further guidelines for avoiding deadlock

deadlock不仅仅出现在有lock的时候，也不局限于两个线程。

避免deadlock的方法可以归结为：不要等待一个可能等待你的线程。

下面的准则能检查并消除，有其他线程等待你的可能性。

**AVOID NESTED LOCKS**

如果需要多个lock，用`std::lock`。

**AVOID CALLING USER-SUPPLIED CODE WHILE HOLDING A LOCK**

用户提供的代码可能会请求另一个lock，进而违反上一个准则。

**ACQUIRE LOCKS IN A FIXED ORDER**

如果要获取两个或更多的lock，并且不能使用`std::lock`一次性获取，次优的方案是在每个线程中以同样的顺序获取。the key is to define the order in a way that's consistent between threads.

**USE A LOCK HIERARCHY**

lock hierarchy提供了一种方法，可以在运行时检查是否遵循lock oerding。如果一段代码已经获得了低层的lock，那么它不允许获得高层的lock。本质上就是为了保证获取lock的顺序。但是这种lock的机制，要求chain中每个mutex的hierarchy value都比前一个低，可能在某些情况下并不实用。

下面是一个hierarchical mutex的例子，

```cpp
class hierarchical_mutex {
  std::mutex internal_mutex;
  unsigned long const hierarchy_value;
  unsigned long previous_hierarchy_value;
  static thread_local unsigned long this_thread_hierarchy_value;

  void check_for_hierarchy_violation() {
    if (this_thread_hierarchy_value <= hierarchy_value) {
      throw std::logic_error("mutex hierarchy violated");
    }
  }

  void update_hierarchy_value() {
    previous_hierarchy_value = this_thread_hierarchy_value;
    this_thread_hierarchy_value = hierarchy_value;
  }

 public:
  explicit hierarchical_mutex(unsigned long value) :
      hierarchy_value(value),
      previous_hierarchy_value(0) {}

  void lock() {
    check_for_hierarchy_violation();
    internal_mutex.lock();
    update_hierarchy_value();
  }

  void unlock() {
    this_thread_hierarchy_value = previous_hierarchy_value;
    internal_mutex.unlock();
  }

  bool try_lock() {
    check_for_hierarchy_violation();
    if (!internal_mutex.try_lock())
      return false;
    update_hierarchy_value();
    return true;
  }
};

thread_local unsigned long hierarchical_mutex::this_thread_hierarchy_value(ULONG_MAX);
```

有这么几个要注意的地方：

**`thread_local`**

这个是这段代码的关键。`thread_local`变量允许你在每个线程中都有一个独立的实例。在namespace的变量、类的static data member和局部变量都可以被声明为`thread_local`，并且拥有thread storage duration。

{ %blockquote% }
When thread_local is applied to a variable of block scope, the storage-class-specifier static is implied if it does not appear explicitly.
{ %endblockquote% }

因此上例中，

```cpp
static thread_local unsigned long this_thread_hierarchy_value;
```

等价于

```cpp
thread_local unsigned long this_thread_hierarchy_value;
```

又因为，C++规定const静态类成员可以直接初始化，其他非const的静态类成员需要在类声明以外初始化，所以有最后的初始化。

**一个线程有多个mutex的情况下，`thread_local`变量是共享的？**

是的，因为它属于类，而不是某个对象。

**三个value会不会在多个线程并发时，出现interleaving修改？**

不会，修改是在internal mutex的lock和unlock之间。

hierarchy mutex本质上说的还是lock的order，其目的就是为了避免多个线程中出现wait cycle，hierarchy mutex是把这种order强制化了。

### Flexible locking with std::unique_lock

`std::unique_lock`提供了比`std::lock_guard`更多的灵活度（构造函数的可传入：`std::defer_lock`、`std::try_to_lock`和`std::adopt_lock`），并且不总是拥有mutex的ownership（locked）。但这两者是有代价的，`std::unique_lock`要占用更多的空间，并且稍微慢一点点，这个可以用源码看出来：

```cpp
    ...
    unique_lock(mutex_type& __m, defer_lock_t) noexcept
    : _M_device(&__m), _M_owns(false)
    { }
    unique_lock(mutex_type& __m, try_to_lock_t)
    : _M_device(&__m), _M_owns(_M_device->try_lock())
    { }
    unique_lock(mutex_type& __m, adopt_lock_t)
    : _M_device(&__m), _M_owns(true)
    {
// XXX calling thread owns mutex
    }
    ...
  private:
    mutex_type* _M_device;
    bool _M_owns; // XXX use atomic_bool
  };
```

`std::unique_lock`需要flag，`_M_owns`来存储mutex的ownership，在unlock、析构等时候，需要根据flag来判断是否要forward to` _M_device`的成员函数来做实际的工作。flag可以通过`std::unique_lock`的`owns_lock()`来获得。

`std::unique_lock`还允许在实例销毁前释放lock，这样就可以在明确知道不再需要lock的时候，进行release，而不必等到销毁时，避免了其他线程额外的等待。

一般来说推荐使用`std::lock_gurad`，但是如果需要额外的灵活度，那就用`std::unique_lock`，例如：

* 延迟锁
* Transfer ownership of lock


### Transferring mutex ownership between scopes

Because `std::unique_lock` instances don't have to own their associated mutexes, the ownership of a mutex can be transferred between instances by moving the instances around.

`std::unique_lock`是movable，但不是copyable的。

其中一个应用是返回一个lock，来transfer ownership给调用函数。

```cpp
std::unique_lock<std::mutex> get_lock() {
    extern std::mutex some_mutex;
    std::unique_lock<std::mutex> lk(some_mutex);
    prepare_data();
    return lk;
}
void process_data() {
    std::unique_lock<std::mutex> lk(get_lock());
    do_something();
}
```

### Locking at an appropriate granularity

lock的粒度是用来描述被一个锁保护的数据的数量。

如果可能，只有在真正访问共享数据的时候才lock a mutex。

一般来说，不要在拥有锁的时候做耗时的工作，尤其是等待其他的lock（就算知道不会deadlock），或者文件I/O（除非真的是需要保护文件的访问）。

`std::unique_lock`很适合用于这样的情况，因为你可以在不需要访问共享数据的时候`unlock()`，需要的时候`再lcok()`。

**关于lock的粒度和拥有锁的时长**

1. 如果只有一个mutex来保护整个数据结构，那么不仅仅很可能会出现更多的竞争，而且减少了lock被held的时间。
2. 对于同一个mutex，如果获得lock以后进行的操作越多，那么lock就会被held越长。

找到一个合适的粒度，不仅仅要看锁住的数据的数量，还要看lock被获得的时长和获得lock的时候做了什么操作。In general, a lock should be held for only the minimum possible time needed to perform the required operations.

还有一个问题就是，由于改变了lock的方式，可能会导致代码语义上的改变，有时这样的改变会导致错误。

## Alternative facilities for protecting shared data

在特殊情况下，有一些比mutex更合适的方式来保护共享数据。

### Protecting shared data during initialization

有时候可能共享数据在创建以后就是只读的，因此只需要在创建的时候进行保护。但是如果使用mutex，仅仅在初始化的时候保护，这是不必要的，并且还会带来不必要的性能损失。

**Double-Checked Locking**

最早接触[DCL](http://www.infoq.com/cn/articles/double-checked-locking-with-delay-initialization)是在设计模式里，单例模式提到过。这个看似高效的方法之所以被骂，就是因为CPU乱序执行可能会导致线程访问没有初始化的对象。

在这里也是类似的，也会带来问题。

```cpp
void undefined_behaviour_with_double_checked_locking() {
    if(!resource_ptr) {
        std::lock_guard<std::mutex> lk(resource_mutex);
        if(!resource_ptr) {
            resource_ptr.reset(new some_resource);
        }
    }
    resource_ptr->do_something();
}
```

这种类型的race condition叫做data race，是undefined behavior。

**lazy initialization with `std::once_flag`和`std::call_once`**

`std::call_once`通常比显式使用mutex会有更低的开销，尤其是当初始化已经完成的时候。

```cpp
std::shared_ptr<some_resource> resource_ptr;
std::once_flag resource_flag;
void init_resource() {
    resource_ptr.reset(new some_resource);
}
void foo() {
    std::call_once(resource_flag,init_resource);
    resource_ptr->do_something();
}
```

当初始化函数是带有函数调用操作符的类的实例的时候，`std::call_once`也支持像`std::thread`和`std::bind()`的用法。

要注意的是，如果类的成员含有不能copy或move的，那么要为它们定义特殊成员函数。

**static**

C++11中，规范了在多线程的情况下初始化static变量。初始化被定义为只发生在一个线程上，直到初始化完成其他线程才能继续。

这可以作为`std::call_once`的替代。

### Protecting rarely updated data structures

用mutex来保护较少更新的数据结构不合适，当没有进行更新的时候，它消除了并发读的可能。

**reader-writer mutex**

允许：

1. 互斥的写或共享；
2. 并发的的读。

C++标准库没有提供这种锁，这里使用的是boost库。这种锁不是万能的，它的性能依赖于处理器的数量和读写线程相对的工作负载。

**`boot::shared_mutex`**

由boost提供。

### Recursive locking

可以在同一个线程中从同一个`std::recursive_lock`的实例获得多次lock。在其他线程获得lock前，当前线程`lock()`了多少次，就必须`unlock()`多少次。

但是并不建议使用`std::recursive_lock`，当持有lock的时候，invariant很可能是broke的。继续lock，意味着要在invariant broken的情况下做操作。

## Reference
1. [Is this an exception safe implementation of swap(multithread)?](http://stackoverflow.com/questions/18047413/is-this-an-exception-safe-implementation-of-swapmultithread)
2. [Are C++11 thread_local variables automatically static?](http://stackoverflow.com/questions/22794382/are-c11-thread-local-variables-automatically-static)
3. [C++ concepts: Lockable](http://en.cppreference.com/w/cpp/concept/Lockable)
4. [双重检查锁定与延迟初始化](http://www.infoq.com/cn/articles/double-checked-locking-with-delay-initialization)
5. C++ Concurrency in Action
