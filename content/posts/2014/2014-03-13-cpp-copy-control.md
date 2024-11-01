title: C++拷贝控制
date: 2014-03-13 20:41:20
categories:
  - cpp
tags:
  - 'c++ primer'
---

# 拷贝、赋值和销毁

## 拷贝构造函数

拷贝构造函数的第一个参数必须是一个引用类型，且几乎总是一个const引用。由于拷贝构造函数在多个情况下会被隐式使用，因此不能是explict的。

```class F {
 public:
  Foo();
  Foo(const Foo&);
}
```

### 合成拷贝构造函数

如果没有定义拷贝构造函数，编译器会定义一个合成拷贝构造函数。不同于合成默认构造函数r，即使自己定义了其它的拷贝构造函数，编译器也会合成一个拷贝构造函数。

合成拷贝构造函数会将**参数的每个非static成员逐个拷贝**到正在创建的对象中。拷贝方式依据成员类型而定，
* 对于类类型，会使用其拷贝构造函数；
* 对于内置类型，会直接拷贝；
* 如果成员有数组类型，合成拷贝构造函数会**逐元素**的拷贝。

合成的函数会被**隐式地声明为内联的**。

### 拷贝初始化

直接初始化会选择**与参数最匹配的构造函数**。拷贝初始化是件右侧运算对象拷贝到正在创建的对象中。

拷贝初始化**通常**使用拷贝构造函数来完成，以下情况会发生拷贝初始化，

* 用`=`定义变量；
* 将对象作为实参传递给一个非引用类型的形参；
* 用一个返回类型为非引用类型的函数返回一个对象；
* 用列表初始化一个数组中的元素或一个聚合类中的成员；
* 某些类类型会对它们所分配的对象使用拷贝初始化（如：容器的insert）。

在进行拷贝初始化时，编译器**可以跳过**拷贝/移动构造函数，直接创建对象，但此时拷贝/移动构造函数必须**存在且可访问**。

如上文所说，拷贝构造函数会被隐式使用，下面是几个例子，

```cpp
class C {
  // ...
};
C c;
vector<C> vc{c};
```

上述代码中，对vector使用列表初始化时，c会被copy两次。1. initializer_list的构造函数会[copy一次](http://stackoverflow.com/questions/20501638/stdvector-init-with-braces-call-copy-constructor-twice)，2. 从initializer_list到设计存储位置还会copy一次。

```cpp
class C {
  // ...
};
void f(C c) {
  C tmp(0);
  tmp = c;
}
void fr(C &c) {
  C tmp(0);
  tmp = c;
}

C obj;

f(obj);
f(C(1));
fr(obj);
fr(C(4));
```

* 第一个f调用，首先会用obj对形参c做拷贝初始化，然后用拷贝赋值，将c赋值给tmp；
* **第二个f调用，这里并没有用临时对象对形参c做拷贝初始化，而是用临时对象C(1)对tmp进行复制。**因为有copy elision。
* 第一个fr调用，直接拷贝赋值，将obj赋值给tmp；
* 第二个fr调用是错误的，问题类似[取临时对象的地址](http://stackoverflow.com/questions/4301179/why-is-taking-the-address-of-a-temporary-illegal)，这里C(4)是rvalue表达式，而fr()需要一个左值作为参数。

## 拷贝赋值运算符

拷贝赋值运算符执行与**析构函数和拷贝构造函数**相同的工作。
如果没有定义拷贝赋值运算符，编译器会定义一个合成拷贝赋值运算符。

### 重载赋值运算符

赋值运算符就是一个名为**operator=**的函数，其参数表示要收费的运算对象。定义为成员函数的运算符，其左侧运算对象就绑定到**隐式的this参数**。返回值通常为**左侧运算对象的引用**。

拷贝赋值运算符参数应为**与所在类相同类型的参数**。

### 合成拷贝赋值运算符

合成拷贝赋值运算符会将右侧运算对象的**每个非static**成员赋予左侧运算对象的相应成员。类似拷贝构造函数逐个拷贝成员，这一工作是由**成员类型的拷贝赋值运算符**完成的。如果是数组类型的成员，则**逐个赋值数组元素**。

如果**自己定义的**拷贝赋值运算符或拷贝构造函数**没有处理成员中的数组**，逐个拷贝/赋值不会发生。

```cpp
class HasPtr {
 public:
  HasPtr(const string &s = string()) : ps(new string(s)), i(0) {}
  HasPtr(const HasPtr &rhs) : ps(new string(*rhs.ps)), i(rhs.i) {}

  string *ps;
  int i;
  int arr[10];
  array<int, 10> sarr;
};

HasPtr a("abc");
a.arr[0] = 232;
a.arr[1] = 232;
a.sarr[0] = 232;
a.sarr[1] = 232;

HasPtr b = a;
HasPtr c;
c = a;
(*b.ps).append("a");
cout << *a.ps << " " << *b.ps << " " << *c.ps << endl;
// abc abca abc
cout << a.arr[0] << " " << b.arr[0] << " " << b.sarr[0] << endl;
// user defined copy constructor
// 232 32627 32627
cout << a.sarr[0] << " " << c.arr[0] << " " << c.sarr[0] << endl;
// synthesized copy-assignment operator
// 232 232 232
```

## 析构函数

销毁对象的**非static**成员。没有返回值，不接受参数（因此不能够被重载）。对于一个给定类，只会有唯一一个析构函数。

销毁顺序按照初始化顺序的**逆序**进行，销毁内置类型成员不需要做什么，销毁类类型成员需要**执行成员自己的**析构函数，销毁内置指针类型的成员**不会delete**所指向的对象。

析构函数在以下情况进行调用，

* 变量离开其作用域；
* 一个对象呗销毁时，其成员也被销毁；
* 容器（标准库容器和数组）被销毁时，其元素被销毁；
* 对动态分配的对象，其指针使用delete；
* 对于临时对象，当创建它的完整表达式结束时被销毁。

### 合成析构函数

合成析构函数函数体为空，当其函数体执行完以后，成员会被自动销毁。析构函数函数体并**不直接销毁成员**，成员是在析构函数函数体之后**隐含的析构阶段**中被销毁的。

## 三/五法则

* 需要析构函数的类也需要拷贝构造函数和拷贝赋值运算符。
* 需要拷贝操作的类也需要赋值操作，反之亦然，但**不必然意味着也需要析构函数**。
* 拷贝操作会带来额外的开销，在拷贝不必须的情况下，应加入移动操作。

## 阻止拷贝

类似`=default`，使用`=delete`可以定义删除的函数。`=delete`必须出现在函数第一次声明的时候，且可以回任何函数指定`=delete`。

如果一个类的析构函数或者一个成员的析构函数是`=delete`，那么将**无法定义该类型的变量或创建该类的临时对象**，可以new但无法delete。

关于合成拷贝控制成员，当**不可能拷贝、赋值或销毁类的成员**时，类的合成拷贝控制成员就被定义为删除的。

# 拷贝控制和资源管理

## 行为像值的类

行为像值的类有自己状态，副本和原对象是完全独立的。

由于赋值操作会**销毁左侧运算对象的资源**，在对如下类定义拷贝赋值运算符时，需要考虑将一个对象赋值给自身的情况。

```cpp
class HasPtr1 {
 public:
  HasPtr1(const string &s = string()) : ps(new string(s)), i(0) {}
  HasPtr1(const HasPtr1 &rhs) : ps(new string(*rhs.ps)), i(rhs.i) {}
  HasPtr1 &operator=(const HasPtr1 &rhs);
  ~HasPtr1() { delete ps; }

 private:
  int i;
  string *ps;
};
```

如果拷贝赋值运算符是这样的，

```cpp
HasPtr1 &operator=(const HasPtr1 &rhs) {
  delete ps;
  ps = new string(*rhs.ps);
  ai = rhs.i;
  return *this;
}
HasPtr1 h;
h = h;
```

将一个对象赋值给自身时，解引用`*rhs.ps`就是错误的，因为ps所指向的对象已经被delete了。因此需要**用一个局部临时对象先保存右侧运算符对象的资源**。

```cpp
HasPtr1 &operator=(const HasPtr1 &rhs) {
  auto newp = new string(*rhs.ps);
  delete ps;
  ps = newp;
  ai = rhs.i;
  return *this;
}
```

## 行为像指针的类

行为像值的类共享状态。

```cpp
class HasPtr2 {
 public:
  HasPtr2(const string &s = string())
      : ps(new string(s)), i(0), use(new size_t(1)) {}
  HasPtr2(const HasPtr2 &rhs) : ps(rhs.ps), i(rhs.i), use(rhs.use) { ++*use; }
  HasPtr2 &operator=(const HasPtr2 &rhs);
  ~HasPtr2() {
    if (--*use == 0) {
      delete ps;
      delete use;
    }
  }

 private:
  string *ps;
  int i;
  size_t *use;
};
```

对于以上类的拷贝构造函数，需要递增右侧运算对象的引用计数，递减左侧运算对象的引用计数。这里同样需要考虑同一个对象给自身赋值的情况，应该先递增右侧运算对象的引用计数，然后递减左侧的并检查，

```cpp
HasPtr2 &operator=(const HasPtr2 &rhs) {
  ++*rhs.use;
  if (--*use == 0) {
    delete ps;
    delete use;
  }
  ps = rhs.ps;
  i = rhs.i;
  use = rhs.use;
  return *this;
}
```

# 交换操作

如果一个类没有定义自己的swap，需要的时候将调用标准库的swap。一般来说，一次swap需要一次copy和两次assign，但这并不是必须要的。如果一个类有动态分配的内存，可以交换指针，而不是既copy又assign。

```cpp
class HasPtr {
 public:
  friend swap(HasPtr &lhs, HasPtr &rhs);
  // ...
}

inline void swap(HasPtr &lhs, HasPtr &rhs) {
  using std::swap;
  swap(lhs.ps, rhs.ps);
  swap(lhs.i, rhs.i);
}
```

要注意的是：

1. HasPtr的成员是内置类型，并没有特定版本的swap，因此上述swap中调用的是标准库的swap；
2. 如果一个**类的成员有自己类型特定的swap**，那么调用std::swap就是错误的，标准库swap会进行不必要的copy。
    ```cpp
    void swap(Foo &lhs, Foo &rhs) {
      using std::swap;
      swap(lhs.h, rhs.h); // 使用HasPtr的swap
    }
    ```
    如果存在类型特定的swap，其匹配程度会优于std中定义的版本。上面的`using std::swap;`**并未隐藏**HasPtr的swap。

## copy and swap

```cpp
HasPtr& HasPtr::operator=(HasPtr rhs) {
  swap(*this, rhs);
  return *this;
}
```

这里的赋值运算符是用传值的方式。swap左侧运算对象和副本，然后销毁副本。

copy and swap天然就是**异常安全**的，因为可能抛出异常的情况就是传值时候的copy，如果此时抛出异常，左侧对象不会被修改。同时保证了**自赋值的正确**，因为是copy。

# move

## lvalue和rvalue

lvalue：有持久的状态，可以取地址。
rvalue：字面值常量或临时对象，不可以取地址。

## rvalue reference

必须绑定到右值，即要求转换的表达式、字面值常量或返回右值的表达式。但**不能直接绑定**到一个左值上。

```cpp
int i = 5;
int &r1 = i * 42; // 错误i*42是右值
const int &r1 = i * 42;
int &&rr1 = i * 42;
```

由于rvalue reference只能绑定到临时对象，因此这个对象，

* 将要被销毁
* 没有其他用户

这意味着使用rvalue reference可以自由地**接管**所引用对象的资源。

```cpp
int i = 5;
int &&rr1 = i * 42;
int &&rr2 = rr1; // 错误
```

## 右值引用类型变量

进行右值引用后，得到的**右值引用类型变量是左值**。

```cpp
int i = 32;
int &&rr1 = i * 32;
cout << &i << endl; // 0x71d77768be28
cout << &rr1 << endl; // 0x71d77768be2c
int &&rr2 = rr1; // 错误rr1是左值
```

要将右值引用绑定到一个左值，应该显示地转换或move，

```cpp
int i = 5;
int &&rri = static_cast<int &&>(i);
int &&rri1 = std::move(i);
// int &&rri2 = i;
```

使用move后，不能对移后源对象的值做任何假设，**不能使用移后源对象的值**。**除了对rri赋值或销毁外**，不能再使用它。

# 移动构造函数和移动赋值运算符

## 移动构造函数

参数是一个右值引用，且**任何额外的参数**都必须有默认实参。完成移动后，必须保证**销毁源对象是无害的**。一旦完成移动，源对象必须**不能再指向被移动的资源**，资源所有权已归属新创建的对象。

```cpp
StrVec::StrVec(StrVec &&s) noexcept : elements(s.elements), first_free(s.first_free), cap(s.cap) {
  s.elements = s.first_free = s.cap = nullptr;
}
```

移动构造函数通常**不分配**任何新内存，因此通常不抛出异常。为避免标准库**为了处理可能抛出异常而做的额外工作**，一种通知标准库的方法是指明`noexcept`。

### 移动操作和异常

* 虽然移动操作通常不抛出异常，但是抛出异常是允许的。
* 标准库能够对异常发生是其自身的行为提供保障。

```cpp
vector<StrVec> vs;
// reallocate vecotr
std::move(vs[i]);
// ...
```

* 上面的代码中，如果在reallocate时，**移动了部分元素**后抛出异常，那么问题就会发生，源vector已经改变，但是新空间有的元素还不存在。
* 如果vector使用拷贝构造函数且发生了异常，那么不会影响源vector。

基于上面两点，**除非vector知道**元素类型的移动构造函数不会抛出异常，否则在reallocate时，就必须使用拷贝构造函数（即前面所说的，*为了处理可能抛出异常而做的额外工作*）。如果希望在类似reallocate 的情况下使用移动而非拷贝，就必须**显式的告诉标准库**移动构造函数可以安全使用。

## 移动赋值运算符

移动赋值运算符执行与**析构函数和移动构造函数**相同的工作。如果不抛出异常，则应该标记为`noexcept`。

```cpp
StrVec &operator=(StrVec &&rhs) noexcept {
  if (this != &rhs) { // 处理自赋值
    free();
    elements = rhs.elements;
    first_free = rhs.first_free;
    cap = rhs.cap;
    rhs.elements = rhs.first_free = rhs.cap = nullptr;
  }
  return *this;
}
```

这里必须check是否是同一对象，因为**此右值可能是move调用返回的结果**，再者不能在使用右侧运算对象的资源之前就释放左侧运算对象的资源。

## 移后源对象必须可析构

从一个对象移动数据并不会销毁此对象，但有时在移动操作完成后，源对象会被销毁，因此需确保移后源对象必须可析构。

移动操作还应保证对象仍是**有效的**，即可以安全地为其**赋予新值**或可以**安全地使用**而**不依赖其当前值**。

移动后，源对象的值是没有保证的，不应依赖移后源对象中的数据。

# 合成移动操作

## 何时定义

```cpp
struct X {
  int i;
  string s;
};

struct hasX {
  X mem;
};

X x, x2 = std::move(x); // 合成移动构造函数
hasX hx, hx2 = std::move(hx); // 合成移动构造函数
```

只有当一个类**没有定义任何自己版本的拷贝控制成员**，且类的**每个非static数据成员都可以移动构造或移动赋值**时，编译器才会合成移动构造函数和移动赋值运算符。

## 何时删除

```cpp
struct Y {
  Y(const Y &y) {}
  int i;
  string s;
};

struct hasY {
  hasY() = default;
  hasY(hasY &&) = default;
  Y mem;
};
hasY hy, hy2 = std::move(hy); // call to implicitly-deleted default constructor of 'hasY'
```

移动操作永远**不会隐式定义**为delete。但如果**显式地**要求编译器生成=default的移动操作，且编译器**不能移动所有成员**时，移动操作会被定义为delete。

定义了一个移动构造函数或移动赋值运算符的类**必须也定义**拷贝操作，否则这些成员**默认定义为delete**。

## 如果未定义

```cpp
class Foo {
 public:
  Foo() = default;
  Foo(const Foo &) { cout << "copied" << endl; }
};
Foo foo;
Foo foo1(foo); // copied
Foo foo2(std::move(foo1)); // copied
```

如果没有移动构造函数，就算试图调用move来移动，对象也会被拷贝。

这里与上面的hasY不同，hasY的移动构造函数是delete的（由于显式地要求生成，但编译器无法生成）。而这里的仅仅是未定义，函数匹配保证该类型的对象会被copy。

## copy and swap again

```cpp
class Hp {
 public:
  Hp() = default;
  Hp(const Hp &) { cout << "copied" << endl; }
  Hp(Hp &&) { cout << "moved" << endl; }
  Hp &operator=(Hp rhs) {
    cout << "assign" << endl;
    return *this;
  }
};

Hp hp1;
Hp hp2 = hp1; // copied，拷贝构造函数
Hp hp3 = std::move(hp1); // moved，移动构造函数
cout << "---" << endl;
hp1 = hp3; // copied assign，拷贝赋值运算符
hp2 = std::move(hp3); // moved assign，移动赋值运算符
```

这里除了拷贝构造函数，还有移动构造函数和赋值运算符。对于之前未定义移动构造函数的情况下，调用赋值运算符，初始化形参时**总是进行拷贝**。

现在拷贝初始化依赖于实参的类型，

* 左值->使用**拷贝构造函数**进行初始化，赋值运算符为**拷贝赋值运算符**
* 右值->使用**移动构造函数**进行初始化，赋值运算符为**移动赋值运算符**

从而单一的赋值运算符，实现了两种功能。

## copy and swap idiom

上面的将拷贝赋值运算符和移动赋值运算符“合并”到一起的方式叫做copy and swap idiom。如果按照上面的方式实现了拷贝赋值运算符和移动赋值运算符，就**不能**再单独写两个拷贝赋值运算符和移动赋值运算符。

而两种实现拷贝赋值运算符和移动赋值运算符的**效率**是有区别的（C++ Primer 5th Chinese Edition, Exercise 15.53）。

```cpp
#define NDEBUG

class HasPtr3 {
  friend void swap(HasPtr3&, HasPtr3&);
  friend bool operator<(const HasPtr3& lhs, const HasPtr3& rhs);

 public:
  HasPtr3(const std::string& s = std::string())
      : ps(new std::string(s)), i(0) {}
  HasPtr3(const HasPtr3& hp) : ps(new std::string(*hp.ps)), i(hp.i) {
#ifndef NDEBUG
    std::cout << "copied" << std::endl;
#endif
  }
  HasPtr3(HasPtr3&& hp) noexcept : ps(hp.ps), i(hp.i) {
    hp.ps = nullptr;
#ifndef NDEBUG
    std::cout << "moved" << std::endl;
#endif
  }

  // = 4s
  // mixed 4.8s
  HasPtr3& operator=(const HasPtr3& rhs) {
    auto newp = new std::string(*rhs.ps);
    delete ps;
    ps = newp;
    i = rhs.i;
#ifndef NDEBUG
    std::cout << "copy assigned" << std::endl;
#endif
    return *this;
  }

  // move 0.7s
  // mixed 4.8s
  HasPtr3& operator=(HasPtr3&& rhs) noexcept {
    if (this != &rhs) {
      delete ps;
      ps = rhs.ps;
      i = rhs.i;
      rhs.ps = nullptr;
#ifndef NDEBUG
      std::cout << "move assigned" << std::endl;
#endif
    }
    return *this;
  }

  // = 6s
  // move 2.5s
  // mixed 8.8s
  // HasPtr3& operator=(HasPtr3 rhs) {
  // swap(*this, rhs);
  // #ifndef NDEBUG
  // std::cout << "assigned" << std::endl;
  // #endif
  // return *this;
  // }

  ~HasPtr3() { delete ps; }

 private:
  std::string* ps;
  int i;
};

inline void swap(HasPtr3& lhs, HasPtr3& rhs) {
  using std::swap;
  swap(lhs.ps, rhs.ps);  // swap the pointers, not the string data
  swap(lhs.i, rhs.i);    // swap the int members

#ifndef NDEBUG
  std::cout << "swapped" << std::endl;
#endif
}
```

测试代码如下，这里最好**不要**把对象的创建放到循环中去，**每次构造和销毁的开销**会使得两种拷贝和移动的区别不太明显，

```cpp
HasPtr3 hp;
HasPtr3 hp1;
HasPtr3 hp2;
auto t0 = high_resolution_clock::now();
for (int i = 0; i < 100000000; ++i) {
  hp = hp1;
  hp = std::move(hp2);
}
auto t1 = high_resolution_clock::now();
cout << duration_cast<milliseconds>(t1 - t0).count() << "ms" << endl;

```

结果如下（VMware 12, Archlinux x64, Intel i7-2620m），

|                                              | `hp = hp1;` | `hp = std::move(hp2);` | mixed |
|----------------------------------------------|-------------|------------------------|-------|
| `HasPtr3& operator=(const HasPtr3& rhs)`     | 4s          | N/A                    | 4.8s  |
| `HasPtr3& operator=(HasPtr3&& rhs) noexcept` | N/A         | 0.7s                   | 4.8s  |
| `HasPtr3& operator=(HasPtr3 rhs)`            | 6s          | 2.5s                   | 8.8s  |

可看出copy and swap idiom的效率是不如分开写好。

# 移动迭代器

一个移动迭代器通过改变给定迭代器的解引用运算符的行为来**适配**此迭代器。对移动迭代器**解引用**生成的是一个**右值**。

```cpp
auto newe = uninitialized_copy(make_move_iterator(elements), make_move_iterator(cap), newb);
```

通过调用`make_move_iterator`可将一个普通迭代器转换为一个移动迭代器。上面的代码中，传递给`uninitialized_copy`的是一个移动迭代器，解引用后得到的是右值，因此`uninitialized_copy`将使用移动构造函数来构造元素。

标准库**不保证**哪些算法适用于移动迭代器。只有确认对象在传递给函数后**不再访问**，才能将移动迭代器传递给算法。

# 右值引用和成员函数

类似构造函数和赋值运算符，成员函数同样可以提供拷贝版本和移动版本。

```cpp
void push_back(const X&); // 拷贝
void push_back(X&&); // 移动
```

拷贝版本接受能够**转换**为类型X的**任何对象**。使用`const X&`是因为拷贝操作**不应该改变该对象**。

而移动版本接受**非const右值**，对于非const右值是精确匹配。从源对象移动数据时，显然需要更改源对象，所以是`X&&`。

## 右值和左值引用成员函数

对于赋值运算符，为了**强制左侧运算对象**是一个左值，可以类似const，在参数列表后使用引用限定符，引用限定符可以是`&`或`&&`，

```cpp
Foo &operator=(const Foo&) &; // 只能向可修改的左值赋值
```

const改变了this指针的类型，指明了this是指向常量的指针，这里类似，引用限定符说明了this**可以指向**一个**左值还是右值**。const和引用限定符只能用于**非static**成员函数。

## 重载和引用函数

* 当定义const成员函数是，可以根据有无const，定义两个重载版本。

    ```cpp
    Foo sorted();
    Foo sorted() const;
    ```

* 当定义有引用限定符的成员函数时，如果定义**两个或两个以上**具有相同**名字和参数列表**的成员函数，就必须对**所有重载函数*都加上引用限定符。

    ```cpp
    Foo sorted(Comp*) &&;
    Foo sorted(Comp*) const; // 错误
    Foo sorted(Comp*) const &;
    ```
