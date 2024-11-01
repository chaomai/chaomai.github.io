title: C++泛型算法
date: 2014-02-23 14:10:17
categories:
  - cpp
tags:
  - 'c++ primer'
---

# 泛型算法

* 一般泛型算法不直接操作容器，而是运行于迭代器之上，由迭代器来进行操作；
* 迭代器令算法不依赖于容器，但是某些算法使用的操作需要元素的类型；
* 算法不会直接改变底层容器的大小；
* 算法可能改变元素的值或移动元素，但不会直接添加或删除元素。

# 按使用元素的方式分类

## 只读算法

### accumulate

`accumulate`的第三个参数类型决定了函数中使用那个加法运算符以及返回值的类型。

* 如果这个类型不支持+运算符，则会发生编译错误；
* 如果元素类型与这个类型不匹配，且能够类型转换，无论哪个类型宽窄，只会发生**元素类型->第三个参数类型**。

### equal

按元素比较，第二个序列**至少与第一个序列一样长**。

如果两个序列类型分别为，`vector<const char*>`和`list<const char*>`，则比较的是地址，不是字符串的内容。

```cpp
vector<const char *> vcc{"ab", "bc", "cd"};
list<const char *> lcc{"ab", "bc", "cd"};

// 由于编译器优化，vcc和lcc实际上共享了字面值常量，故下面的地址相同。
cout << equal(vcc.begin(), vcc.end(), lcc.begin()) << endl;
// 1
cout << static_cast<void *>(const_cast<char *>(vcc.front())) << endl;
// 0x40c951
cout << static_cast<void *>(const_cast<char *>(lcc.front())) << endl;
// 0x40c951

const char a[3][3] = {"ab", "bc", "cd"};
const char b[3][3] = {"ab", "bc", "cd"};
vector<const char *> vcc1{begin(a), end(a)};
list<const char *> lcc1{begin(b), end(b)};

cout << equal(vcc1.begin(), vcc1.end(), lcc1.begin()) << endl;
// 0
cout << static_cast<void *>(const_cast<char *>(vcc1.front())) << endl;
// 0x40c908
cout << static_cast<void *>(const_cast<char *>(lcc1.front())) << endl;
// 0x40c911
```

## 写容器元素的算法

这类算法并不检查写操作。由于算法不会改变底层容器的大小，因此必须保证**目的位置迭代器开始**序列足够容纳要写入的元素。

`copy`返回的是目的位置迭代器递增后的值。

## 重排容器元素的算法

`unique`“移除”了**相邻重复**元素，把不重复的元素移动到了序列前面，并非删除。返回不重复元素范围末尾的下一个迭代器。

# 定制操作

## 谓词

某些算法需要进行元素间的比较，如果需要使用与定义行为不同的比较，或者元素类型未定义`<`运算符，则需要通过提供谓词，重载算法的默认行为。

谓词是可调用的表达式，返回结果是一个能用着条件的值，分为，

* 一元谓词；
* 二元谓词。

序列中的元素作为实参传入谓词，因此需要满足函数匹配规则。

## lambda表达式

是可调用对象，callable object有，

* 函数
* 函数指针
* 重载了函数调用运算符的类
* lambda表达式
* bind创建的对象

使用lambda，要注意的是，

* lambda必须使用尾置返回，可以忽略参数列表和返回类型（忽略时，从代码中推断）；

    ```cpp
    auto f = []{ return 1; };
    ```

* lambda不能有默认参数；
* 对于lambda所在函数体的**非static局部变量**，只能使用在**捕获列表**中捕获后，才能使用；
* 对于**局部static变量和lambda所在函数体之外声明的名字**，可以**直接使用**。

如果lambda捕获列表为空，那么lambda可以转换为函数指针。

```cpp
auto f = [](int a, int b) { return a + b; };
// f自动转换为pointer
int (*pf)(int, int) = f;
cout << pf(1, 2) << endl;
```

### lambda捕获和返回

定义lambda时，编译器生成一个与lambda对应的未命名类类型。用auto定义一个用lambda初始化的变量时，就定义了一个相应的未命名类类型的对象。lambda的数据成员在lambda对象创建时被初始化。

类似函数的参数传递，捕获方式有，

每种方式都可以进行，

* 值捕获

    被捕或的变量的值是在lambda创建时拷贝，而不是像函数调用时才拷贝。能使用值捕获的前提是**变量可拷贝**。

* 引用捕获

    `&`表示以引用的方式捕获。
    引用捕获与返回引用有相同的问题和限制，必须保证引用的对象在**lambda执行时存在**，且在执行时是所期望的（可能在被捕或后和执行前，引用的对象的值改变了）。如果函数返回lambda，则不能包含局部非static变量的捕获。

按是否显示列出希望使用的变量，可分为，

* 显式捕获

    `[v1, ...]`或`[&v1, ...]`

* 隐式捕获

    * 值捕获，`[&]`
    * 引用捕获，`[=]`

* 混合使用显式和隐式捕获

    当混合使用显式和隐式捕获时，显示捕获的变量必须使用**与隐式捕获不**同的方式。

    * `[&, identifier_list]`

        任何**隐式捕获**的变量都采用**引用捕获**，**identifier_list**采用**值捕获**的方式，且identifier_list中的名字**不能使用`=`**。

    * `[=, identifier_list]`

        任何**隐式捕获**的变量都采用**值捕获**，**identifier_list**采用**引用捕获**的方式，且identifier_list中的名字**必须使用`&`**。

        ```cpp
        int a = 1;
        int b = 2;
        int c = 3;

        auto l = [=, &c](){};
        // auto l = [=, c](){}; 错误
        ```

### 可变lambda

 默认情况下，**值捕获的变量**，lambda不会改变被捕获变量的值（并非改变原始变量），如果需要改变，加入`mutable`。

```cpp
int a = 1;

auto l2 = [a]() mutable {
  ++a; // 如果无mutable，则错误
  return a;
};
cout << a << endl; // 1， 并非改变原始变量
cout << l2() << endl; // 2
```

引用捕获无此限制，能够更改依赖于被引用变量是否为`const`。

### lambda的返回值

默认情况下，如果一个lambda包含return之外的任何语句，编译器假定此lambda返回void。

## bind

由于`find_if`接受的是一个一元谓词，因此含有两个形参的函数是不可用的。

`bind`生成一个新的可调用对象，可看做一个通用的函数适配器。

```cpp
auto newCallable = bind(callable, arg_list)
```

arg_list可包含`_n`，表示`newCallable`的参数，参数的类型就是callable中`_n`处的类型。其中n代表了占位符`_n`在`newCallable`的位置，调用`newCallable`时，`_n`处的参数最终会传递到callable中`_n`相应的位置。

`_n`定义在`placeholders`的namespace中，使用时需，

```cpp
using std::placeholders::_1;
...
// 或
using namespace std::placeholders;
```

默认情况下，**不是占位符的参数**是被**拷贝**到bind返回的可调用对象中的。如果想传递引用，就必须显式的指明，使用`ref`。`ref`返回的对象包含给定的引用，是**可拷贝**的。类似的还有`cref`，返回`const`引用。

# 额外的迭代器

除每个容器的定义的迭代器外，还有以下几种，

## 插入迭代器

是一种迭代器适配器，对插入迭代器赋值时，该迭代器**调用容器的操作**来进行插入。`*`和前置后置`++`会直接返回迭代器，**并不会修改**。

插入迭代器有以下几种，

* `back_inserter`

    创建使用`push_back`的迭代器，始终在尾部插入。

* `front_inserter`

    创建使用`push_front`的迭代器，始终在首部插入。

* `inserter`

    创建使用`insert`的迭代器，始终在迭代器it指定位置前插入。插入后it还是指向一开始指定的位置。

## 流迭代器

### `istream_iterator`

读取输入流，必须指定迭代器将要读写的对象类型，且要读取的类型必须支持`>>`（由于是调用`>>`来读取）。

```cpp
istream_iterator<int> int_it(cin);
istream_iterator<int> int_eof; // 默认初始化，尾后迭代器

while (int_it != eof) {
  ...
}

vector<int> v(int_it, eof);
```

当绑定到流时，标准库并不保证迭代器立即从流中读取数据，而是**保证在首次解引用前，完成了数据的读取**。

### `ostream_iterator`

类似`istream_iterator`，使用`<<`写入输出流（类型必须支持`<<`）。**必须**绑定到一个输入流，可指定一个每次写入时都输出的字符。`*`和前置后置`++`会直接返回迭代器，**并不会修改**。

```cpp
ostream_iterator<int> out(cout);
// ostream_iterator<int> out(cout, " "); 字面值常量或指向C风格的字符串

out = 5; // 类型必须与out定义的类型兼容
```

## 反向迭代器

也是一种迭代器适配器。只能从一个支持++和--的迭代器来定义反向迭代器。

```
    cbegin()                                     cend()
     |                                            |
    [], [], [], [], [], [], [], [], [], [], [], []
   |                                             |
crend()                                         crbegin()

cbegin()                 rit.base()         cend()
 |                           |                |
[], [], [], [], [], [], [], [], [], [], [], []
                         |                   |
                        rit               crbegin()

 [crbegin(), rit)和[rit.base(), cend())指向相同的范围。
```

可以对反向迭代器调用`base()`来得到普通的迭代器。`base()`得到的是相邻的位置。

## 移动迭代器

一个移动迭代器通过改变给定迭代器的解引用运算符的行为来**适配**此迭代器。对移动迭代器**解引用**生成的是一个**右值**。

```cpp
auto newe = uninitialized_copy(make_move_iterator(elements), make_move_iterator(cap), newb);
```

通过调用`make_move_iterator`可将一个普通迭代器转换为一个移动迭代器。上面的代码中，传递给`uninitialized_copy`的是一个移动迭代器，解引用后得到的是右值，因此`uninitialized_copy`将使用移动构造函数来构造元素。

# 泛型算法的结构

## 迭代器分类

按算法所要求的迭代器操作，可将迭代器分为下面几类，**除了输出迭代器外**，高层类别的迭代器支持低层类别迭代器的所有操作。C++标准指明了泛型和数值算法的每个迭代器参数的**最小类别**。

1. 输入迭代器

    **只读，不写；单遍扫描，只能递增**

    支持，

    * `==`，`!=`
    * 前置后置`++`，可能导致所有其他指向流的迭代器**失效**，不能保证输入迭代器的状态可以保存下来，即只能单遍扫描
    * `*`，只能在赋值运算的**右侧**
    * `->`

    例子：`find`，`accumulate`。

2. 输出迭代器

    **只写，不读；单遍扫描，只能递增**

    支持，

    * 前置后置`++`
    * `*`，只能在赋值运算的**左侧**

    例子：用作目的位置的迭代器，如`copy`。

3. 前向迭代器

    **可读写；多遍扫描，只能递增**

    支持，

    * 所有输入和输出迭代器的操作

    例子：`replace`，`forward_list`的迭代器。

4. 双向迭代器

    **可读写；多遍扫描，可递增递减**

    支持，

    * 所有前向迭代器的操作
    * 前置后置`--`

    例子：`reverse`，除了`forward_list`的迭代器，其他标准库容器的迭代器都符合双向迭代器。

5. 随机访问迭代器

    **可读写；多遍扫描，支持全部迭代器的运算**

    支持，

    * `<`，`<=`，`>`，`>=`
    * 和整数的`+`，`-`，`+=`，`-=`
    * 两个迭代器的`-`
    * `iter[n]`，等价于`*(iter[n])`

    例子：`sort`，`array`、`deque`、`string`和`vector`的迭代器。

## 算法形参模式

* `alg(beg, end, other, args);`
* `alg(beg, end, dest, other, args);`
* `alg(beg, end, beg2, other, args);`
* `alg(beg, end, beg2, end2, other, args);`

向输出迭代器`dest`写入数据的算法都假设，目标位置有足够的空间。
接受单独`beg2`的算法假定从`beg2`开始的序列与`[beg, end)`所表示的范围至少一样大。
