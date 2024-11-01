title: C++重载运算与类型转换
date: 2014-03-20 20:01:13
categories:
  - cpp
tags:
  - 'c++ primer'
---

# 运算符重载

重载的运算符是具有**特殊名字的函数**，除函数调用运算符外，其他重载运算符**不能含有默认实参**。

* 一个运算符函数，或者**类的成员**，或者**至少含有一个类类型的参数**。也就是说不能为内置类型重载运算符。
* 只能重载**已有的**运算符。
* 一个重载的运算符，其优先级和结合律与对应的内置运算符**保持一致**。
* 由于使用重载的运算符本质上是**函数调用**，因此对象求值顺序的规则**无法应用**到重载的运算符上。尤其是`&&`，`||`和`,`，两个运算对象**总是会**被求值。
* 通常情况下，不应该重载`&&`，`||`，`&`和`,`，运算符。
* 一般来说，提供了某个重载运算符，也应该提供**与此运算符相关的**一系列运算符，如：算术运算符->对应的复合赋值运算符。

## 成员或者非成员

* `=`，`[]`，`()`和`->`运算符**必须是成员。
* 复合赋值运算符**一般来说应该是**成员，但非必须。
* 改变对象状态或与给定类型密切相关的运算符，应该是成员。如：递增、解引用。
* 具有对称性的运算符可能转换任意一端的运算对象，通常应该是非成员。如：算术、相等性、关系和位运算符。如果想提供**含有类对象的混合类型表达式**，运算符必须是非成员函数。

    ```cpp
    string s = "world";
    string u = "hi" + s; // 如果是成员函数，则"hi".operator+(s)，错误
    ```

对于友元要注意，在类内部虽然有友元声明，但这**并非真正意义上的函数声明**，因此在类外部还需要有函数声明。

# 输入输出运算符

## <<

* `ostream &operator<<(ostream &os, const Sales_data &item);`
* 必须是**非成员函数**，否则左侧运算对象将是类的一个对象
    就算要是某个类的成员，也只能是istream或ostream的，然而并不能为标准库中的类添加成员，因此只能是非成员函数
* 不应该打印换行符

## >>

* `istream &operator>>(istream &is, Sales_data &item);`
* 必须处理输入可能失败的情况

    ```cpp
    istream& operator>>(istream &is, Sales_data &item) {
      double price = 0.0;
      is >> item.bookNo >> item.units_sold >> price;
      if (is) {
        item.revenue = price * item.units_sold;
      } else {
        item = Sales_data();
      }
      return is;
    }
    ```
* 有时需要标识**流的条件状态**

# 算术和关系运算符

* 通常把算术和关系运算符定义为非成员函数，使得左侧或右侧的运算对象可以转换

    ```cpp
    string s = "world";
    string u = "hi" + s; // 如果是成员函数，则"hi".operator+(s)，错误
    ```
* 这些运算符一般不需要改变运算对象的状态，所以形参是常量引用
* 一般使用**复合赋值来实现**算术运算符

## ==

* 相等运算符应该具有**传递性**
* 定义了`==`，也应该定义`!=`

## 关系运算符

关系运算符应该，

1. 定义顺序关系，且与关联容器中对关键字的要求一致
2. Define a relation that is consistent with `==` if the class has both operators. In particular, if two objects are `!=`, then one object should be `<` the other.

如果存在唯一一种逻辑可靠的<语义，才考虑定义<运算符。如果类同时还包含==，则当且仅当<的定义和==产生的结果一致时才定义<运算符。

## =

* 必须为成员函数
* **必须先释放**当前内存空间
* 如果不是拷贝赋值运算符和移动赋值运算符，则**不必检查自赋值**的情况。

## []

* 必须为成员函数
* 返回元素的引用
* 通常定义const和非const版本

# 递增递减运算符

由于改变了对象的状态，一般应为**成员函数**，且应该**同时定义**前置版本和后置版本。

```cpp
// 返回递增或递减后对象的引用
StrBlobPtr &operator++();
StrBlobPtr &operator--();

// 返回对象的原值
StrBlobPtr operator++(int); // 形参不会被使用，仅仅是和前置版本进行区分
StrBlobPtr operator--(int);

p.operator--(0); // 调用后置版本的
p.operator--(); // 调用前置版本的
```

# 成员访问运算符

```cpp
string &StrBlobPtr::operator*() const {
  auto p = check(curr, "deference past end");
  return (*p)[curr];
}

string *StrBlobPtr::operator->() const { return &this->operator*(); }
```

* `->`**必须是类成员**，`*`非必需，但**通常也是**。
* 定义为const是因为获取一个元素并不会改变对象的状态。
* 虽然可以让解引用返回任何想要的值或打印（不建议），但箭头运算符**必须有成员访问的含义**。重载箭头运算符时，可以改变的是箭头从哪个对象中获取成员。

# 函数调用运算符

函数调用运算符**必须是**成员函数。如果类定义了调用运算符，那么该类的对象叫做函数对象。调用函数对象实际上是在运行重载的调用运算符。

## lambda是函数对象

编写lambda后，编译器将生成一个**未命名类的未命名对象**。这个类中有一个重载的函数调用运算符，且默认情况下这个成员是const（lambda不能改变捕获的变量），除非lambda被声明为mutable。

* 引用捕获
    由程序确保lambda执行时所引用的对象确实存在，生成的类中无须保存为数据成员。

* 值捕获
    生成的类中需建立对象的数据成员，同时创建构造函数，用捕获的变量来初始化数据成员。

lambda产生的类中，**不包含默认构造函数、赋值运算符和默认析构函数**，默认拷贝和默认移动构造函数视捕获的数据成员类型而定。

# 标准库定义的函数对象

标准库规定的函数**对于指针同样适用**。直接比较两个无关的指针将产生未定义的行为，但通过标准库定义的函数对象来比较是**定义良好的**。

关联容器使用`less<key_type>`对元素排序，因此可以直接定义一个指针的set或map，而无须声明`less`。

# function

C++中的可调用对象多种有，它们的类型是不同的，

* 函数和函数指针
    类型由返回值类型和实参类型决定
* lambda表达式
    每个lambda有唯一的未命名类类型
* bind创建的对象
* 重载了函数调用运算符的类

类型不同的可调用对象**可能共享**同一种调用形式。调用形式指明了**调用返回的类型**以及**传递给调用的实参类型**。一种调用形式对应一个函数类型。

定义`funcion`类型时，需要指明调用形式。

```cpp
  map<string, function<int(int, int)>> binops = {{"+", plus<int>()},
                                                 {"-", minus<int>()},
                                                 {"*", multiplies<int>()},
                                                 {"/", divides<int>()},
                                                 {"%", modulus<int>()}};

```

# 重载、类型转换

如果构造函数只接受一个实参，则它实际上定义了转换为此类型的**隐式转换**机制，这种构造函数叫做转换构造函数。

**转换构造函数**和**类型转换运算符**共同定义类类型转换，也叫用户定义的类型转换。

编译器**一次只能执行一个**用户定义的类型转换。

## 隐式类型转换运算符

* `operator int() const;`
    从类类型转换为int
* 无显示的返回类型和形参
* 返回一个对应类型的值
* 必须定义为成员函数
* 通常不应该改变转换对象的内容，一般被定义为const

```cpp
class S {
 public:
  S(int i = 0) : val(i) { cout << "construct" << endl; }
  S(const S &s) : val(s.val) { cout << "copy" << endl; }
  S(S &&s) : val(s.val) { cout << "move" << endl; }
  S &operator=(const S &s) {
    cout << "copy assign" << endl;
    val = s.val;
    return *this;
  }
  S &operator=(S &&s) {
    cout << "move assign" << endl;
    val = s.val;
    return *this;
  }
  operator int() const {
    cout << "conversion->int" << endl;
    return val;
  }

 private:
  size_t val;
};

S s; // construct
s = 4; // construct， move assign
s + 3; // conversion->int
s - s; // conversion->int, conversion->int
```

S虽然没有定义`-`，但隐式转换为int，可以执行内置的`-`。

虽然编译器一次只能执行一个用户定义的类型转换，但**隐式的用户定义类型转换**可以至于一个**标准（内置）类型转换之前或之后**。

```cpp
// double->int->S
s = 3.14; // construct， move assign
// S->int->double
s + 3.14; // conversion->int
```

## 显式类型转换运算符

`explicit operator int() const;`

```cpp
class S {
 public:
  S(int i = 0) : val(i) {}
  explicit operator int() const { return val; }

 private:
  size_t val;
};

S s;
s + 3; // 错误
static_cast<int>(s) + 3;

if (s) {
  // ...
}
```

但是当表达式**作为条件**时，编译器会将显式类型转换**自动应用**于它（仅仅是自动应用`explicit operator bool() const`，以转换为bool）。

```cpp
class S {
 public:
  S(int i = 0) : val(i) {}
  explicit operator bool() const { return val; }

 private:
  size_t val;
};

S s;
if (s) {
}
```

## 避免有二义性的类型转换

必须确保类类型和目标类型之间**只存在唯一一种**转换方式。**无法**通过使用强制类型转换来解决二义性问题，强制类型转换也会面临二义性问题。

多重转换路径可能由于，

* 两个类提供相同的类型转换
* 类定义了一组类型转换，它们的转换源（或者转换目标）类型本身可以通过其他类型转换联系在一起，由于**所有算术类型转换的级别都一样**，选择转换序列时会有**多个转换序列**，将会导致二义性。

    ```cpp
    class S {
     public:
      S();
      operator int() const;
      operator double() const;
    };

    S s;
    int a = s; // 精确匹配
    float b = s; // 两个“可行函数”
    ```

> 如果已经定义了一个转换为算术类型的类型转换，**不要**再定义**接受算术类型的重载运算符**。在不定义以后，如果用户需要使用这样的运算符，则类型转换操作会转换此类型的对象，然后使用**内置的运算符**。

### 重载函数与转换构造函数

当调用重载的函数时，如果两个或多个类型转换都提供了可行的匹配，则这些类型转换一样好。

### 重载函数与用户定义的类型转换

当调用重载的函数时，如果两个或多个类型用户定义的转换都提供了可行的匹配，则这些类型转换一样好。此时，不考虑任何可能出现的标准类型转换级别。

**只有**当重载函数能通过同一个类型转换函数得到匹配时（所有可行函数都请求同一个用户定义的类型转换），才考虑标准类型转换级别。

## 函数匹配与重载运算符

表达式中运算符的候选函数集包括**成员函数**和**非成员函数**。

如果对同一个类既提供了转换目标是算术类型的类型转换，也提供了重载的运算符，则将会遇到**重载运算符和内置运算符的二义性问题**。
