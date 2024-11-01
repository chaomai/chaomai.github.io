title: C++动态内存
date: 2014-03-03 18:31:05
categories:
  - cpp
tags:
  - 'c++ primer'
---

静态内存：存储局部static对象、类static数据成员和定义在函数之外的变量。
static对象：使用之前分配，程序结束时销毁。

栈内存：保存定义在函数内部的非static对象。
栈对象：仅在定义的程序块运行时才存在。

动态内存（free store或heap）：存储动态分配的对象，需要**显示地销毁**，分配和销毁由`new`和`delete`完成。

# 智能指针

## shared_ptr

最安全的分配和使用动态内存的方法是调用`make_shared`，返回指向在动态内存分配的对象的`shared_ptr`。`make_shared`类似`emplace`，使用**参数来构造**指定类型的对象，如果没有参数，则进行值初始化。

当进行copy或assign时，**每个**`shared_ptr`会记录有多少个其他`shared_ptr`指向相同的对象。可看作`shared_ptr`有reference count，

* 当发生以下情况时，count递增；
    1. copy或assign；
        ```cpp
        shared_ptr<string> p = make_shared<string>("hello"); // 1
        auto q(p); // 2
        ```
    2. 作为参数传给一个函数；
    3. 作为函数的返回值；

* 当发生以下情况时，count递减；
    1. 给`shared_ptr`赋予一个新的值；
        ```cpp
        shared_ptr<int> r = make_shared<int>(42); // 1
        r = q; // ++q指向对象的ref count，--r指向对象的ref count；销毁r原来指向对象
        ```
    2. `shared_ptr`被销毁；

count的递减由`shared_ptr`的析构函数完成，如果count**变为0**，`shared_ptr`会释放所管理的对象。

在某个scope中，只要能够使用`shared_ptr`，那么它的引用计数**至少为1**。

### 操作

```cpp
p.get() // 返回内置的指针
p.use_count() // 返回引用计数
p.unique() // 若引用计数为1，则返回true；否则false
p.reset() // 将p置空，若p是唯一指向对象的，则释放此对象
p.reset(q) // 令p指向内置指针q，若p是唯一指向对象的，则释放此对象
p.reset(q, d) // 同上，释放q时调用d
```

使用`new`和`delete`会使得类对象的copy、assign和destroy**不能依赖任何默认定义**。

### 自定义释放操作

默认情况下，shared_ptr指向的是动态内存，因此被销毁时，默认调用delete。可以自定义释放操作，提供其他的deleter。deleter的参数必须为**该shared_ptr的内置指针类型**。

## 直接管理动态内存

### new

默认情况下，new的对象是**默认初始化**的。也可以使用值初始化的方式来初始化new的对象（圆括号+参数），还可以使用列表初始化，以及值初始化（空括号）。

### 自动推断类型

可以使用auto从initializer来推断将要分配的对象类型，由于编译器**需要从initializer来获得类型**，因此圆括号中仅能有一个initializer，

```cpp
auto p1 = new auto(obj);
auto p2 = new auto{a, b, c}; // 错误
auto p3 = new auto{a}; // 错误
```

### const对象

和其他const对象相同，必须初始化。

```cpp
const string *pcs = new const string; // 调用默认构造函数
const int *pci = new const int(1024); // 只能显示初始化
```

### bad_alloc

如果内存不足，new失败，就会抛出`bad_alloc`，但可以告知不抛出。

```cpp
int *p1 = new int;
int *p2 = new (nothrow) int; // 失败则返回nullptr
```

### delete

传递给delete的必须是指针，且必须指向动态分配的内存，或是一个`nullptr`。如果是动态分配的内存，或释放同一个指针多次，行为未定义。对const动态对象，销毁的方法也是一样的。

```cpp
// 使用clang++ 3.7编译
int i = 5;
int *pi = &i;
delete pi; // segmentation fault

int *pi2 = nullptr;
delete pi2; // ok

double *pd = new double(3);
double *pd1 = pd;
delete pd;
cout << pd << endl;
// pd不会被置为nullptr，空悬指针
delete pd; // core dumpped

const int *pci = new const int(1024);
delete pci;
```

可以在delete后手动赋值为`nullptr`。但也仅仅只解决了`pd`的问题，多个指针指向同一个内存区域时，仍然有问题，`pd1`仍然指向原内存区域，还是空悬指针。

## shared_ptr和new

可以用new返回的指针来初始化`shared_ptr`。由于接受智能指针的构造函数是explicit的，因此必须使用**直接初始化**。

```cpp
shared_ptr<int> p(new int(1024));
shared_ptr<int> p1 = new int(1024); // 错误，此语句首先需要在int*和shared_ptr之间做隐式转换，然后再把临时的shared_ptr拷贝给p1。
```

`shared_ptr`定义了`get`函数，可以获得内置指针，指向`shared_ptr`管理的对象。通过这种方式得到的指针**不能被delete**，必须保证代码不会delete的情况下，才能使用get。

```cpp
// 使用clang++ 3.7编译
shared_ptr<int> p(make_shared<int>(4));
int *q = p.get();
{
  shared_ptr<int> sq(q);
  cout << *sq << endl; // 4
  delete q; // double free or corruption
}
cout << *p << endl; // 错误，p指向的内存已被销毁
// double free or corruption
```

上述代码在内部的scope中手动删除了p指向的内存，当这个scope结束时，sq被销毁，那部分内存**还会被shared_ptr销毁一次**。编译时不会报错，但运行时出现double free or corruption。
**就算没有delete**，内部的scope结束，那部分内存被销毁，这段代码结束时，又一次被销毁，同样也会有double free or corruption。

## 智能指针和异常

无论是函数正常结束或者发生异常，局部对象都会被销毁。智能指针被销毁时，如果引用计数为0，则释放内存。但new得到的内存不会被自动释放，如果有指向这块内存的指针，只有指针会被销毁。

## unique_ptr

unique_ptr**拥有**指向的对象。没有类似make_shared的函数，只能将其绑定到new返回的指针上。也是必须使用直接初始化。

### 操作

除了将被销毁的unique_ptr外，**不支持copy和assignment**。

```cpp
u = nullptr // 释放u指向的对象，并置空
u.release() // 放弃对指针的控制权，返回指针并将u置空
u.reset() // 类似shared_ptr，只是无需判断引用计数
u.reset(q)
u.reset(nullptr)
```

### 自定义释放操作

不同于shared_ptr，unique_ptr在重载deleter时，需要**提供deleter的类型**。重载unique_ptr的deleter，会影响到unique_ptr的类型和如何构造或reset该类型的对象。

```cpp
unique_ptr<objT, delT> p(new objT, fcn);
```

## weak_ptr

* **不控制**所指向对象的生命周期；
* weak_ptr指向shared_ptr管理的对象；
* 将一个weak_ptr绑定到一个shared_ptr**不会改变shared_ptr的引用计数**；
* 一旦shared_ptr被销毁，所指对象就被释放。

### 操作

```cpp
w = p // p可以是weak_ptr或shared_ptr
w.reset() // 将w置空
w.use_count()
w.expired() // 若w.use_count()为0，返回true，否则false
w.lock() // 若w.expired()为true，返回空的shared_ptr，否则返回指向w的对象的shared_ptr
```

# 动态数组

分配动态数组的类必须**定义自己的版本**的操作来管理**拷贝，复制以及销毁**。

## new数组

```cpp
int *pia = new int[32];
```

`pia`中的元素是进行默认初始化的。但此时pia并**不是一个数组**类型的对象，只是一个数组元素类型的指针，因此不能够调用begin和end（它们**使用数组的维度**来得到首元素和尾后元素指针），也不能使用for。

### 初始化

new的数组和单个对象一样，默认情况下，new的数组是**默认初始化**的。可以对数组中的元素进行值初始化和列表初始化，也和单个对象一样。

```cpp
string *psa = new string[10]();
auto *psa1 = new string[10]("abd", "abc", ...); // 不能再这里提供initializer，因此不能用auto
int *pia = new[10]{0, 1, 2, 3, 4, 5}; // 剩下的元素进行值初始化
```

如果new失败，类似bad_alloc，这里会抛出bad_array_new_length。

### new空数组

这样做是合法的，得到的是一个合法的非空指针，相当与数组的尾后指针，不能解引用。

```cpp
int *p = new int[0]; // 合法，但不可以解引用
```

### 释放动态数组

```cpp
int *p = new int[10];
delete [] p;
delete p; // 未定义
```

释放时，按逆序销毁。p还可以为nullptr。

### 智能指针和动态数组

标准库提供了一个管理new分配的数组的unique_ptr，但此unique_ptr**不支持成员访问运算符**。unique_ptr被销毁时，会自动使用`delete []`。

```cpp
unique_ptr<T[]> u
unique_ptr<T[]> u(p)
u[i]
```

如果使用shared_ptr来管理，则必须**提供自定义的删除器**。如果没有提供，则shared_ptr会默认调用delete，行为未定义。

```cpp
shared_ptr<int> sp(new int[10], [](int *p) { delete [] p; });
*(sp.get() + 5) = 2; // shared_ptr未定义下标运算符，且智能指针不支持指针算术运算
```

## allocator

```cpp
class C {
 public:
  C(int a, int b) : a_(a), b_(b) {}

 private:
  int a_;
  int b_;
};

C *const pc = new C[10]; // 错误
C *const pc1 = new C[5]{1, 2, 3, 4, 5}; // 错误
```

new把**内存分配和对象构造**组合在了一起，可能造成外的开销；同时若类没有默认构造函数，则不能够分配动态数组。

allocator**分离**内存分配和对象构造，避免不必要的开销。所分配的内存是原始的，未构造的。

```cpp
allocator<C>::size_type n = 10;
allocator<C> alloc;
auto const p = alloc.allocate(n); // C *const p1 = alloc.allocate(10);
auto q = p;
alloc.construct(q++, 1, 2); // 类似make_shared
cout << p->a_ << " " << p->b_ << endl;
cout << q->a_ << " " << q->b_ << endl;  // 错误，q指向的内存未构造

while (q != p) {
  alloc.destroy(--q);
}

alloc.deallocate(p, n); // 大小应和allocate时的一样
```

### 拷贝和填充未初始化内存

下列操作所需的内存是由`allocate`分配的，而**不是系统分配**的，因此`alloc_b`指向的内存必须足够大。

```cpp
uninitialized_copy(b, e, alloc_b); // 返回最后一个构造的元素之后的位置
uninitialized_copy_n(b, n, alloc_b); // 返回最后一个构造的元素之后的位置
uninitialized_fill(alloc_b, alloc_e, t);
uninitialized_fill_n(alloc_b, n, t);
```
