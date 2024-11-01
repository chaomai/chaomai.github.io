title: C++ Copy Elision
date: 2016-03-05 13:03:30
categories:
    - cpp
tags:
    - 'copy elision'
---

在写代码是发现拷贝构造函数有时候没有调用，想起C++ Primer中提到过

> the compiler can omit calls to the copy constructor.

后来查到是发生了copy elision。

首先有那么一个类定义，其中静态成员c是对象编号。

```cpp
class C {
 public:
  C() {
    name_ = c;
    ++c;
    cout << "c" << name_ << ": " << __func__ << endl;
  }
  C(const int name) : name_(name) {
    cout << "c" << name_ << ": " << __func__ << endl;
  }
  C(const C &rhs) : name_(c++) {
    cout << "copy from c" << rhs.name_ << ", c" << name_ << ": " << __func__
         << endl;
  }
  C &operator=(const C &rhs) {
    cout << "copy assign from c" << rhs.name_ << ", c" << name_ << ": "
         << __func__ << endl;
    return *this;
  }

  int name_ = 0;
  static int c;
};

int C::c = 10000;
```

然后是有下面的几个函数定义和调用，

```cpp
void f(C c) {
  C tmp();
  tmp = c;
}
C f() { return C(); }

C c1(1);
f(c1);
f(C(2));
C c = f();
```

以上三个f的调用分别输出什么？

下面是结果，

```
c1: C
copy from c1, c10000: C
c10001: C
copy assign from c10000, c10001: operator=
-----
c2: C
c10002: C
copy assign from c2, c10002: operator=
-----
c10003: C
```

第一个没什么好说的，首先用`c1`对形参`c`做拷贝初始化，接着`tmp`进行默认初始化，然后用拷贝赋值，将`c`赋值给`tmp`。

第二个的结果就有点怪了，为什么`C(2)`得到的临时对象直接进行了赋值，而不首先初始化形参`c`？而第三个，为什么返回的临时对象一次拷贝都没发生？

因为在这两种情况中发生了copy elision（[1](http://stackoverflow.com/questions/12953127/what-are-copy-elision-and-return-value-optimization)，[2](http://stackoverflow.com/questions/8451212/passing-temporary-object-as-parameter-by-value-is-copy-constructor-called)，[3](https://en.wikipedia.org/wiki/Copy_elision)，[4](http://cstdlib.com/tech/2014/07/12/nrvo-and-copy-elision/)）。**Copy elision**是一种优化手段，满足特定条件时会发生，当传入的参数是rvalue的时候，无需进行额外的拷贝，直接使用源对象。**RVO**，全称叫[return value optimization](https://en.wikipedia.org/wiki/Return_value_optimization)，编译器会让调用函数在其栈上分配空间，被调函数返回值处的临时对象会在这块内存上构造，进而避免了return时临时对象的拷贝，是copy elision常见形式。根据返回的对象是否是临时的，有**named return value optimization**和**return value optimization**。

Copy elision和rvo即使在有可观察的到的side-effects时，也会发生，是[As-if rule](https://en.wikipedia.org/wiki/As-if_rule)的例外中的一种。Dave Abrahams写过pass by value的一系列[文章](https://web.archive.org/web/20140205194657/http://cpp-next.com/archive/2009/08/want-speed-pass-by-value/)。Ayman B. Shoukry在[这里](https://msdn.microsoft.com/en-us/library/ms364057(v=vs.80).aspx#nrvo_cpp05_topic3)讨论了nrvo的局限（multiple return points和conditional initialization）。

clang++和g++可以用`-fno-elide-constructors`控制是否开启优化。关闭优化后，输出的结果就和期待的一致了，

```
c1: C
copy from c1, c10000: C
c10001: C
copy assign from c10000, c10001: operator=
-----
c2: C
copy from c2, c10002: C
c10003: C
copy assign from c10002, c10003: operator=
-----
c10004: C
copy from c10004, c10005: C
copy from c10005, c10006: C
```

stackoverflow的这个[答案](http://stackoverflow.com/questions/12953127/what-are-copy-elision-and-return-value-optimization)，给出了Standard reference和发生copy elision以及return value optimization常见的例子，下面是搬运例子，

* nrvo

```cpp
class Thing {
 public:
  Thing();
  ~Thing();
  Thing(const Thing&);
};
Thing f() {
  Thing t;
  return t; // optimization
}
Thing t2 = f();
```

* rvo

```cpp
class Thing {
 public:
  Thing();
  ~Thing();
  Thing(const Thing&);
};
Thing f() {
  return Thing(); // optimization
}
Thing t2 = f();
```

* pass a temporary object by value

```cpp
class Thing {
 public:
  Thing();
  ~Thing();
  Thing(const Thing&);
};
void foo(Thing t); // optimization

foo(Thing());
```

* exception is thrown and caught by value

```cpp
struct Thing{
  Thing();
  Thing(const Thing&);
};

void foo() {
  Thing c;
  throw c; // optimization
}

int main() {
  try {
    foo();
  }
  catch(Thing c) {}
}
```
