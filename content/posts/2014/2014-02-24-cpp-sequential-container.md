title: C++顺序容器
date: 2014-02-24 14:53:06
categories:
  - cpp
tags:
  - 'c++ primer'
---

首先介绍顺序容器操作基本相同的部分，然后分别是每种容器要注意的地方。

# 定义和初始化

* `C c`
    默认构造函数
* `C c1(c2)`，`C c1 = c2`
    c1初始化为c2的copy。c1和c2必须是相同的容器类型，且保存相同类型的元素。
* `C c{a, b, c, ...}`，`C c = {a, b, c, ...}`
    列表初始化
* `C c(b, e)`

除array外，还有以下两种，
* `C seq(n)`
* `C seq(n, t)`

## `C c1(c2)`和`C c(b, e)`

`C c1(c2)`要求两个容器必须是相同的类型，且元素类型也是相同的；但`C c(b, e)`就只需要元素类型可以**转换**为要初始化的容器的元素类型即可。

# 赋值和swap

所有容器都支持赋值`=`，赋值后，左边容器的元素为右边容器元素的copy，且大小与右边容器相同。

## swap

* `swap`会交换两个容器的元素，两个容器必须有**相同的**类型；
* 通常来说`swap`只是交换了容器内部的数据结构，但也有例外，对array进行操作时，会真正交换它们的元素；
* `swap`完成以后，**除string外**，指向容器的迭代器、引用和指针都不会失效。

## assign

* `seq.assign(b, e)`
* `seq.assign(il)`
* `seq.assign(n, t)`

要注意的有，

* 参数非常像初始化的；
* `assign`不适用于关联容器的array；
* 可以从一个**不相同但相容的**类型assign。

# 大小

除了forward_list，每个容器都支持`size()`、`empty()`和`max_size()`；forward_list只支持`empty()`和`max_size()`。

# 运算符

每种容器都支持`==`和`!=`。

除了无序关联容器，所有容器都支持`>`，`>=`，`<`和`<=`。

# 操作

## push...和insert

`push_back`，`insert`和`push_front`放入容器的是元素的copy。

## emplace...

`emplace_front`，`emplace`和`emplace_back`分别对应`push_back`，`insert`和`push_front`。不支持push...或insert的容器，也不支持相应的`emplace`。

与`push_back`，`push_front`和`insert`放入元素的copy不同，`emplace`会见参数传递给元素类型的构造函数。

```cpp
class C2 {
 public:
  C2() = default;
  explicit C2(int a) : a_(a) {}
  C2(int a, int b) : a_(a), b_(b) {}

 private:
  int a_ = 0;
  int b_ = 0;
};

vector<C2> cs2;
// 由于是explicit，不存在从int到C2的隐式转换
// cs2.push_back(1);
// cs2.push_back(); // 错误
// cs2.push_back(1, 2); // 错误

cs2.push_back(C2());
cs2.push_back(C2(1));
cs2.push_back(C2(1, 2));

// 调用了C2的构造函数，对应以三个push_back
cs2.emplace_back();
cs2.emplace_back(1);
cs2.emplace_back(1, 2);
```

`emplace`在容器中直接构造元素，由于参数是传递给元素的构造函数，因此实参的类型必须和构造函数匹配。

## 访问元素

可用以下方法访问顺序容器的元素：

* `c.back()`
* `c.front()`
* `c[n]`
* `c.at(n)`

下标操作和at实际上就是进行随机访问，因而只能用于支持随机访问的顺序容器（string、vector、deque和array）。两个随机访问中，只有at能够保证安全的随机访问，下标越界时，会抛出out_of_range异常。

## 删除元素

* `c.pop_back()`
* `c.pop_front()`
* `c.erase(p)`
* `c.erase(b, e)`
* `c.clear()`

`pop_back`和`pop_front`返回`void`，`erase`返回一个迭代器，位置为最后一个被删除元素的下一个。

如果`b`和`e`相等（即使都为`c.end()`），那么不会删除任何元素。

# 容器适配器

适配器是一种机制，使得某种事物的行为看起来像另外一个事物。容器、迭代器和函数都有适配器。

每个适配器有两个构造函数，

* `A a`
* `A a(c)`，拷贝容器c来初始化a

顺序容器适配器有，

* stack，默认情况下基于queue实现；
* queue，默认情况下基于queue实现；
* priority_queue，默认情况下基于vector实现；

一般来说，用于构造适配器的容器是有限制的，

* 能添加和删除元素；
* 访问尾元素；

具体到每个适配器，

* stack
   `push_back`，`pop_back`，`back`
* queue
    `push_back`，`push_front`，`back`，`front`
* priority_queue
    `front`，`push_back`，`pop_back`，随机访问

# vector

## capacity和size

vector支持快速随机访问，元素是连续存储的。这意味着添加元素时，需要移动已有的元素，以保证连续的存储。如果没有空间容纳新元素，则需要分配新的内存，并把已有元素从旧的位置移动到新的位置。为了减少内存的分配和释放的代价，分配策略一般为实在是没法存时，才获取新的内存。只要没有操作使得vector的capacity不够，就不会重新分配内存。

以下几个操作是管理容器大小的，

* `c.shrink_to_fit()`
    请求把capacity减少为size。这里仅仅是一个请求，shrink_to_fit**不保证**退还内存。
* `c.capacity()`
    在不重新分配内存的情况下，最多能存储的元素数目。
* `c.reserve(n)`
    分配至少能容纳n个元素的内存。n如果小于等于当前capacity，那么什么都不发生。

要注意的是，reserve和resize不同，reserve改变（至少是变大）的是capacity，size并未变化；而resize改变了size的同时，capacity（如果n大于当前的capacity）也有可能改变。

```cpp
void t23() {
  vector<int> v1(10, 10); // size: 10 capacity: 10
  vector<int> v2(10, 10); // size: 10 capacity: 10
  v1.resize(15); // size: 15 capacity: 20
  v2.reserve(15); // size: 10 capacity: 15
}
```

## 迭代器

添加元素的情况下，
* 如果添加前`capacit = size`，那么会导致内存重新分配，从而vector相关的迭代器、引用或指针都会失效；
如果不发生内存分配，有可能会发生元素移动（插入中间位置），插入位置之后的迭代器、引用或指针都会失效。

对于删除元素，不会发生内存重新分配，被删除元素前的迭代器、引用或指针都还有效。

# deque

## 迭代器

添加元素的情况下，
如果插入到**首尾**之外的位置，都会导致迭代器、引用或指针失效；

删除元素的情况下，
* 如果在**首尾**之外的位置删除，都会导致迭代器、引用或指针失效；
* 如果删除了**尾元素**，**尾后迭代器也会失效**，但其他的迭代器、引用或指针不受影响。

# string

## 操作

string也支持上述提及的大部分操作，但同样有例外，且某些操作有特殊局限，

* `swap`会导致相关的迭代器、引用或指针失效；
* 与vector一样，front相关的操作不支持；
    * 不支持`push_front`和`emplace_front`；
    * 不支持`pop_front`;
* `insert`、`erase`、`assign`和`replace`重载函数，p323和p324；
    * 额外的`insert`、`erase`、`assign`；
    * `append`在末尾插入；
    * `replace`，等价于`erase`+`insert`。
* 搜索，p325和p326；
    * 返回值均为`string::size_type()`，是一个`unsigned`类型，如果找不到，这返回`string::npos`；
    * `s.find(args)`，返回第一个匹配args的下标；
    * `s.rfind(args)`，返回最后一个匹配args的下标；
    * `s.find_first_of(args)`，返回args中任何一个字符**首次出现在args中**的下标；
    * `s.find_last_of(args)`；
    * `s.find_first_not_of(args)`，返回首个**不在args中**字符的下标；
    * `s.find_last_not_of()`。
* compare同样有多个重载，p327；
* 数值转换，p328；
    * `to_string(val)`；
    * `sto...`；
        * string中的第一个非空白字符必须是数值中可能出现的字符，即+或-，或数字，数字可以是0x或0X开头表示的十六进制数；
        * 如果是转换为浮点数的，开头可以为.，且包含e或E表示指数；
        * 如果是转换为整型的，根据不同的基数，可以有字母；
        * 如果不能转换，则抛出`invallid_argument`；
        * 如果得到的数值无法用任何类型表示，则抛出`out_of_range`。
