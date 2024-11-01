title: "C++ Concurrency in Action (5) - The C++ memory model and operations on atomic types"
date: 2015-07-18 18:41:54
description: The C++ memory model and operations on atomic types的笔记。
categories:
    - concurrency
tags:
		- atomic
    - reading
    - cpp
    - cpp11
    - 'c++ concurrency in action'
---

# Memory model basics

two aspects:

* the basic structural aspects;
* the concurrency aspects.

## Objects and memory location

C++程序中所有的数据都是由object组成，object是"a region of storage"。一个对象存储在一个或多个**memory location**。

每个memory location，

* 要么是，一个标量的一个对象（或子对象）；
* 要么是，相邻bit fields的序列。（虽然相邻的bit fields是不同对象，它们仍然算作相同的memory location，除非用长度为0的bit fields隔开。）

## Modification orders

C++程序中的每个对象都定义了一个modification order，它由程序中的所有线程对这个对象的write组成， starting with the object's initialization。

在绝大多数情况下，这个order在每次运行的时候都是不同的，但对于一个给定的执行，所有的线程都必须agree on the order。如果不使用原子类型，你就必须提供有效的同步来使得这些线程都agree on the modification order of each variable。但是线程并没有必要agree on the relative order of separate objects。

# Atomic Operations and types in C++

一个atomic operation是**indivisible operation**，要么完成，要么不完成。

## The standard atomic types

`is_lock_free()`：给定类型的操作是直接由atomic instructions完成，还是由编译器和库提供的内部锁完成。

`std::atomic_flag`没有提供`is_lock_free()`成员函数。因为在这个类型上的操作required to be lock-free，一旦有了这个lock-free的类型，就能够以它为基础，进而实现所有其他的atomic类型。

在大多数平台上，所有内置类型的atomic变种都**应该**是lock-free的，但这并不是必须的。

要注意的是，由于历史的原因，在有的平台，atomic类型指的不一定是`std::atomic<>`的specialization（如：`atomic_bool`和`std::atomic<bool>`）。如果混用，就可能导致不兼容的情况出现。

**The alternative names for the standard atomic types and their corresponding `std::atomic<>` specializations.**

|Atomic type|Corresponding specialization|
|-|-|
|`atomic_bool`|`std::atomic<bool>`|
|`atomic_char`|`std::atomic<char>`|
|`atomic_schar`|`std::atomic<signed char>`|
|`atomic_uchar`|`std::atomic<unsigned char>`|
|`atomic_int`|`std::atomic<int>`|
|`atomic_uint`|`std::atomic<unsigned>`|
|`atomic_short`|`std::atomic<short>`|
|`atomic_ushort`|`std::atomic<unsigned short>`|
|`atomic_long`|`std::atomic<long>`|
|`atomic_ulong`|`std::atomic<unsigned long>`|
|`atomic_llong`|`std::atomic<long long>`|
|`atomic_ullong`|`std::atomic<unsigned long long>`|
|`atomic_char16_t`|`std::atomic<char16_t>`|
|`atomic_char32_t`|`std::atomic<char32_t>`|
|`atomic_wchar_t`|`std::atomic<wchar_t>`|

**The standard atomic `typedefs` and their corresponding built-in `typedefs`**

|Atomic `typedef`|Corresponding Standard Library `typedef`|
|-|-|
|`atomic_int_least8_t`|`int_least8_t`|
|`atomic_uint_least8_t`|`uint_least8_t`|
|`atomic_int_least16_t`|`int_least16_t`|
|`atomic_uint_least16_t`|`uint_least16_t`|
|`atomic_int_least32_t`|`int_least32_t`|
|`atomic_uint_least32_t`|`uint_least32_t`|
|`atomic_int_least64_t`|`int_least64_t`|
|`atomic_uint_least64_t`|`uint_least64_t`|
|`atomic_int_fast8_t`|`int_fast8_t`|
|`atomic_uint_fast8_t`|`uint_fast8_t`|
|`atomic_int_fast16_t`|`int_fast16_t`|
|`atomic_uint_fast16_t`|`uint_fast16_t`|
|`atomic_int_fast32_t`|`int_fast32_t`|
|`atomic_uint_fast32_t`|`uint_fast32_t`|
|`atomic_int_fast64_t`|`int_fast64_t`|
|`atomic_uint_fast64_t`|`uint_fast64_t`|
|`atomic_intptr_t`|`intptr_t`|
|`atomic_uintptr_t`|`uintptr_t`|
|`atomic_size_t`|`size_t`|
|`atomic_ptrdiff_t`|`ptrdiff_t`|
|`atomic_intmax_t`|`intmax_t`|
|`atomic_uintmax_t`|`uintmax_t`|

要注意的是：

1. 标准的atomic类型**不是copyable和assignable**的；

	> 因为这些操作总是涉及到两个对象，必须从一个中read，然后write到另一个，这是两个单独的操作，合起来不可能是atomic。因此就不被允许。

2. 支持assignment from和implicit conversion to对应的内置类型；
3. 赋值操作返回的并不是reference  to  the  object  it's
assigned to，而是the value assigned。

	> 因为如果返回了reference  to atomic variable，那些使用这个变量的代码需要显示的`load()`，实际使用到值的可能是其他线程已经修改过的。

**Memory-ordering**

每个在atomic类型上的操作都有memory-ordering参数，来指定memory-ordering语义。不同的操作可传入不同的参数，操作分为三类：

1. Store
2. Load
3. Read-modify-write

默认是`memory_order_seq_cst`。

## Operations on `std::atomic_flag`

代表一个boolean flag，只能是两种状态：set或clear，并且总是starts clear。

必须用`ATOMIC_FLAG_INIT`来初始化，`std::atomic_flag guard = ATOMIC_FLAG_INIT`。这是唯一需要使用这样的特殊方式来初始化的类型，也是**唯一一个**保证lock-free的。

如果是`static`，那么在首次操作flag的时候初始化。

用于实现spinlock，

```cpp
class spinlock_mutex {
    std::atomic_flag flag;
public:
    spinlock_mutex():
        flag(ATOMIC_FLAG_INIT) {}
    void lock() {
        while(flag.test_and_set(std::memory_order_acquire));
    }
    void unlock() {
        flag.clear(std::memory_order_release);
    }
};
```

## Operations on `std::atomic<bool>`

可用nonatomic bool来初始化，还可以向实例赋nonatomic bool值，**`std::atomic<bool>`可能不是lock-free的！**

```cpp
std::atomic<bool> b(true);
b = false;
```

* write：`store()`；
* read-modify-write：`exchange()`；
* nonmodifying query：`load()`。

```cpp
std::atomic<bool> b;
bool x=b.load(std::memory_order_acquire);
b.store(true);
x=b.exchange(false,std::memory_order_acq_rel);
```

**compare/exchange**

* `compare_exchange_weak()`
* `compare_exchange_strong()`

如果失败，expected value或被更新为original value，都接受两个memory-ordering参数。

有这么几个要注意的地方：

1. 一个failed compare/exchange是不会进行保存的，因此某些memory-ordering语义是不可用的（`memory_order_release`和`memory_order_acq_rel`）。
2. can't supply stricter memory ordering for failure than for success。
3. 如果不为failure提供memory-ordering参数，则**在满足1的情况下**，与success一致。
4. 如果都不提供，则使用默认的`memory_order_seq_cst`。
5. 它们是read-modify-write operation。

*蛋疼的作者啊，很多地方扯到memory-ordering语义，但是总是说“leave to section 5.3...”*

## Operations on `std::atomic<T*>`

`std::atomic<bool>`有的成员函数，这个也有。

`std::atomic<T*>`还提供了pointer arithmetic operations，由`fetch_add()`和`fetch_sub()`实现。

**`fetch_add()`和`fetch_sub()`**

1. 也叫做*exchange-and-add*，它们是atomic read-modify-write operation。
2. 返回的是原始的值，而不是add或sub后的值。

	```cpp
	class Foo{};
	Foo some_array[5];
	std::atomic<Foo*> p(some_array);
	Foo* x=p.fetch_add(2);
	assert(x==some_array);
	assert(p.load()==&some_array[2]);
	x=(p-=1);
	assert(x==&some_array[1]);
	assert(p.load()==&some_array[1]);
	```

## Operations on standard atomic integral types

Only division, multiplication, and shift operators are missing。因为atomic integral types通常作为计数器或位域来使用，如果需要额外的操作，可以在loop中用`compare_exchange_weak()`来实现。

## The `std::atomic<>` primary class template

要将user-defined type用于`std::atomic<>`，UDT必须满足：

1. 必须有trivial copy-assignment operator；

	> * 不能有任何的virtual函数或者virtual基类；
	> * 必须使用编译器生成的copy-assignment operator。

2. 每个基类和非static数据成员必须有trivial copy-assignment operator；

	> * 这可以使得编译器将`memcpy()`或等价的操作用于assignment operation。

3. 这个类型必须是bitwise equality comparable。

	> * 这里接着2，不仅要能够使用`memcpy()`来copy，还要能使用`memcmp()`来比较（以便compare/exchange能工作）。

**为什么要满足？**

1. If **user-supplied** copy-assignment or comparison operators were permitted, this would **require passing a reference to the protected data** as an argument **to a user-supplied function**, thus violating the guideline.
2. 增大了编译器对`std::atomic<UDT>`直接使用atomic instruction的可能，因为编译器可以把UDT看作a set of raw bytes。

**`std::atomic<float> and std::atomic<double>`？**

因为表示的不同，即使相等`compare_exchange_strong()`也会fail。

## Free functions for atomic operations

与原子类型的成员函数相对应，也有相应的非成员函数，大多数前面都会加上`atomic_`。

要注意的地方有：

1. 用含有`_explict`的版本来指定memory ordering；
2. 所有free functions的第一个参数类型都是pointer to atomic objcet(为了C-compatible)；

	> 对于CAS，要么不指定failure memory ordering，要么两个都要指定。

3. 对于`std::atomic_flag`只能，

	* `std::atomic_flag_test_and_set()`
	* `std::atomic_flag_clear()`
	* `std::atomic_flag_test_and_set_explicit()`
	* `std::atomic_flag_clear_explicit()`

4. `std::shared_ptr<>`算是特殊，它非atomic type，但支持load, store, exchange and compare/exchange。这些free functions第一个参数接受`std::shared_ptr<>*`。

# Reference

1. [std::atomic_flag as member variable](http://stackoverflow.com/questions/24437396/stdatomic-flag-as-member-variable)
