title: C++中typedef的使用和类型安全
date: 2015-02-06 23:06:00
description: 传统的`typedef`机制允许对已存在的type提供synonym或者alias，被定义为新引入的alias的类型与被定义为原来类型的变量，完全一样，不会有一丁点行为上的差别，但是这种特性在某些场景下会有缺陷。
categories:
  - programming
tags:
  - cpp
---

# 关于`typedef`

传统的`typedef`机制允许对已存在的type提供synonym或者alias，我们把这种的传统的`typedef`（包括c++11中的alias声明）描述为“透明类型机制”：这种声明引入了新的类型名称，但不是新的类型。被定义为新引入的alias的类型与被定义为原来类型的变量，完全一样，不会有一丁点行为上的差别。

# 问题

但是这种特性在某些场景下会有缺陷。

## 问题1

``` cpp
using score = unsigned;
score penalize(score n) { return n > 5u ? n - 5u : score { 0u }; }

using serial_number = unsigned;
serial_number next_id(serial_number n) { return n + 1u; }
```

使用新的alias可以很明显的了解以上代码的意图，但实际上这样的使用意图在错误使用alias的情况下，是无法保证的，或者说是不可实行的(unenforceable)。

`unsigned`, `next_id`和`serial_number`是可以互换的。`penalize()`所penalize的不一定是`score`，也可以是`serial_number`。这样虽然编译器不会报错，但是代码没有意义。编码的时候如果错误的使用了alias，还会导致难以track的问题。

## 问题2

``` cpp
typedef int sound_id;
sound_id create_sound(...)
void destroy_sound(sound_id id);

typedef int sprite_id;
void destroy_sprite(sprite_id);

sound_id fx = create_sound(...);
destroy_sprite(fx);  // An honest mistake!
```

这里的问题也是类似问题1。或许你可以认为自己能够小心的编码，来保证alias的正确使用，但是这种欺骗自己的想法，并不能100%的保证问题不会发生。If they can happen, they will happen!

## 问题3

这个问题也是类似，但不仅仅会有误用的风险，同时还导致了类型系统的问题。

``` cpp
typedef double X, Y, Z; // Cartesian 3D coordinate types
typedef double Rho, Theta, Phi; // spherical 3D coordinate types

class PhysicsVector {
public:
    PhysicsVector(X, Y, Z);
    PhysicsVector(Rho, Theta, Phi);
    · · ·
}; // PhysicsVector
```

在上述例子中，`typedef`的大量使用实际上破坏了类型系统（问题域中的类型和构造函数中的类型）。笛卡尔坐标系的三个坐标值和球坐标系中的，虽然都是`double`，但明显意义是不一样的。尽管误用编译器不会报错，但是程序是有bug的。

另一个问题是，两个构造函数实际上就是一个，它接受三个`double`类型的参数。但这里的意图是，分别为两种坐标系建立构造函数。

# 解决

## opaque type(clang++ 3.5.1, g++4.8.2和VS2013都不支持)

引入opaque（不透明的） typedef，在发生误用时，由编译器来检查。

opaque typedef定义了一种新的类型，这种新的类型与它的underlying type不同，并且与它的underlying type是可区分的，同时还保证了与它的underlying type的layout compatibility。

## Type tags

``` cpp
template<class Tag, class impl, impl default_value>
class ID
{
public:
    static ID invalid() { return ID(); }

    // Defaults to ID::invalid()
    ID() : m_val(default_value) { }

    // Explicit constructor:
    explicit ID(impl val) : m_val(val) { }

    // Explicit conversion to get back the impl:
    explicit operator impl() const { return m_val; }

    friend bool operator==(ID a, ID b) { return a.m_val == b.m_val; }
    friend bool operator!=(ID a, ID b) { return a.m_val != b.m_val; }

private:
    impl m_val;
};
```

这个方法相当与简单的包装了一下。参数Tag其实并没有在模板里面使用，它的使用方式如下：

``` cpp
struct sound_tag{};
typedef ID<sound_tag, int, -1> sound_id;

struct sprite_tag{};
typedef ID<sprite_tag, int, -1> sprite_id;
```

tag保证了`sound_id`和`sprite_id`是不同的type，换句话说，只要函数声明了不同了type，它们就不会被误用。

## `BOOST_STRONG_TYPEDEF`

如果使用boost库，那么boost提供了`BOOST_STRONG_TYPEDEF`。

``` cpp
BOOST_STRONG_TYPEDEF(int, a);
BOOST_STRONG_TYPEDEF(T, D);
```

这个宏为类型`T`创建了新的类型名`D`。

# 参考

1. [Type safe handles in C++](http://www.ilikebigbits.com/blog/2014/5/6/type-safe-identifiers-in-c)

2. [Toward Opaque Typedefs for C++1Y, v2](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3741.pdf)
