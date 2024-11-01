title: C++关联容器
date: 2014-02-28 18:56:17
categories:
  - cpp
tags:
  - 'c++ primer'
---

关联容器支持普通容器操作，不支持，

* 顺序容器位置相关的操作，`push_back`等；
* 构造函数或插入操作接受一个元素值和一个数量值得操作。

# 定义

定义一个map时必须指定**关键字类型和值类型**，set只需**关键字类型**。

# 初始化

可以用下面的方式来初始化，

* 同类型容器的copy；
* 指定值范围（begin和end）；
* 列表初始化。

# 有序关联容器关键字类型的要求

有序关联容器关键字类型必须定义**元素比较的方法**。默认情况下，使用`<`进行比较。

```cpp
// list的iterator并无<
map<list<int>::iterator, int> ml; // 声明不会报错
list<int> li;
ml[li.begin()] = 0; // 报错
```

对于有序容器，有序容器的关键字必须是**严格弱序**的，可看做“小于等于”。

## 自定义比较操作

用于组织一个容器中元素的**操作的类型**也是容器类型的一部分，如果需要自定义操作，则在定义容器的时候就指明。

```cpp
boo compare(...) { ... }
multiset<Sales_sata, decltype(compare) *> bookstore(compare);
```

创建对象时，提供的操作类型（**函数指针**）必须与尖括号中的类型吻合。规则与函数的const形参和实参的规则一致，忽略top const。

```cpp
auto comp = [](int a, int b) { return false; };
multiset<int, bool (*)(const int, const int)> m(comp);

auto comp1 = [](const int a, const int b) { return false; };
multiset<int, bool (*)(int, int)> m1(comp1);

auto comp2 = [](int* const a, int* const b) { return false; };
multiset<int, bool (*)(int*, int*)> m2(comp2);

auto comp3 = [](int* a, int* b) { return false; };
multiset<int, bool (*)(int* const, int* const)> m3(comp3);

auto comp4 = [](int* a, int* b) { return false; };
multiset<int, bool (*)(const int*, const int*)> m4(comp4); // 错误
```

## pair

模板，接受两个类型名，pair的数据成员将有对应的类型，两个类型不要求一样。

创建对象时，pair的默认构造函数对数据成员进行**值初始化**（vector也可以）。

```cpp
pair<string, size_t> word_count;
```

若函数返回pair，可**对返回值**进行列表初始化，不必显式构造返回值。

# 关联容器的操作

map中，每个元素就是一个pair对象，由于关键字不可变，因此pair的**关键字部分是const**。set的关键字也是`const`。

```cpp
map<string, int>::value_type v1; // pair<const string, int>
map<string, int>::key_type v2; // string
map<string, int>::map_type v3; // int
```

## 关联容器的迭代器

对关联容器迭代器解引用，可得到容器的`value_type`的引用。

对于set，虽然set定义了`iterator`和`const_iterator`，但是都不能改变set中的元素。

当迭代器遍历一个map，multimap，set或multiset时，按关键字升序遍历。

## 关联容器和算法

由于关键字是`const`，因此不能用于修改或重排容器的算法（都需要向元素写入值）。只可用于读取元素的算法。

## 添加元素

`insert`和`emplace`可以对关联容器进行插入，

```cpp
c.insert(v)
c.emplace(args)
// map和set返回pair<iterator, bool>，iterator指向有此关键字的元素，bool说明是否元素是否已经存在，即是否插入成功。multimap和multiset总是进插入，只返回bool。

c.insert(b, e)
c.insert(li)
// 返回void

c.insert(p, v)
c.emplace(p, args)
// p指明了从哪里开始新元素的存储位置
```

## 删除元素

关联容器有三个版本的erase，

```cpp
c.erase(k)
// 删除所有key为k的元素，返回size_type，表明删除的数量

c.erase(p)
// 返回被删除元素后的迭代器

c.erase(b, e)
// 返回e
```

## map的下标

由于set并无关联值，下标操作对set无意义，故set不支持。multimap和multiset可能存在多个与某个key关联的值，故也不支持。

下标操作返回`mapped_type`，是左值。如果关键字不在map中，下标操作会，

1. **创建一个元素并插入**，关联值将进行**值初始化**；
2. 提取元素并赋值。

注意，

1. 与vector和string不同，map的下标操作和解引用返回的类型（`mapped_type`和`value_type`）不一样；
2. 如果元素不存在，`at`并不会创建，而是抛出`out_of_range`；
3. 下标操作可能会改变map，对const的map无法使用；
4. 对于const的map，只要`at`不修改元素，就可以用。
    ```cpp
    const map<string, int> m{{"hello", 1}};
    cout << m.at("hello") << endl;
    cout << m["hello"] << endl; // 错误，即使是普通访问
    m.at("hello") = 1; // 错误
    ```

# 无序关联容器

使用hash function和`==`来组织元素，用`hash<key_type>`类型的对象生成每个元素的hash值，有序关联容器的操作可以用于无序容器。

标准库为**内置类型（包括指针类型）和部分标准库类型（包括string和智能指针）类型**定义了hash。但不能直接把自定义类型作为key来定义无序容器，可以

1. 提供自己的hash模板版本；
2. 定义hash function和`==`运算符。

```cpp
size_t hasher(const Sales_data& sd) {
  return hash<string>() (sd.isbn());
}

bool eqOp(const Sales_data& lhs, const Sales_data&rhs) {
  return lhs.isbn() == rhs.isbn();
}

unordered_multiset<Sales_data, decltype(hasher)*, decltype(eqOp)*> set(42, hasher,  eqOp);
```
