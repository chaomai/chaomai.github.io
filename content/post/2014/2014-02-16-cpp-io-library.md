---
title: C++IO库
date: 2014-02-16 22:10:25
categories:
  - cpp
tags:
  - 'c++ primer'
---

# `iostream`

## 条件状态

`strm::iostate`是**机器相关**的位类型，用来标识流错误状态，有4个值，`strm::badbit`，`strm::failbit`，`strm::eofbit`和`strm::goodbit`。可用`ios::bad()`，`ios::fail()`，`ios::eof()`和`ios::good()`来得知各个flag是否置位；用`ios::rdstate()`来获取流的当前状态。

```bash
badbit: 0001
failbit: 0100
eofbit: 0010
goodbit: 0000
```

不同的错误会导致不同的flag被置位，

* eof -> eofbit，failbit
* fail -> failbit
* bad -> badbit，且fail()返回true

用`ios::clear()`和`ios::setstate()`可以对flag进行置位，两者区别在于，`ios::clear()`会先将原先的所有flag先复位，而后者不会，

```cpp
void ios::setstate (iostate state) {
  clear(rdstate() | state);
}
```

## 输出缓冲

每个输出流都管理一个缓冲区。

### 缓冲刷新

即数据真正写到输出设备或文件，会有以下原因，

* 程序正常结束，main函数return；
* 缓冲区满，需要刷新才能写入后续数据；
* 使用操纵符；
    * `endl`：换行并刷新缓冲区；
    * `flush`：刷新缓冲区，不附加任何额外字符；
    * `ends`：附加一个空字符，刷新缓冲区。
* `unitbuf`和`nounitbuf`；
    使用`unitbuf`后，接下来的每次写操作都进行一次flush；使用`nounitbuf`返回正常的缓冲方式。
* 关联输入流和输出流。
    关联一个输入流到输出流时，从输入流读取数据，会先刷新关联的输出流，`cin`和`cout`是被关联的。使用`tie`进行关联。

```cpp
// tie的两个重载版本
// 两个版本都返回已经tied ostream，如果没有，则返回nullptr
ostream* os = cin.tie();
(*os) << "tie version 1" << endl;

ostream* old_os = cin.tie(nullptr);
ostream* null_os = cin.tie();
(*old_os) << "tie version 2" << endl;
cout << boolalpha << (old_os == &cout) << "\n" << (null_os == nullptr) << "\n";
// true true

char ch;
// 此时上面的两个true并不会输出
cin >> ch;
cout << ch << endl;
```

# `fstream`

```flow
op1=>operation: iostream
op2=>operation: fstream

op2(right)->op1
```

继承自`iostream`，

* 接受一个`iostream`类型引用或指针参数的函数，可用一个对应的`fstream`类型来调用；
* 支持`iostream`的所有操作。

`fstream`也有自己特有的行为和操作，

* `fstream::fstream(s)`：创建对象并打开文件`s`；
* `fstream::open()`：打开文件，并与`fstream`对象关联；
* `fstream::close()`：关闭与`fstream`对象关联的文件；
* `fstream::is_open()`：与`fstream`对象关联的文件是否成功打开，且未关闭；
* 对于已打开的`fstream`对象，再次`open()`会失败；
* `open()`失败->failbit；
    ```cpp
    ifstream ifs(...);
    if (ifs) {
    ...
    }
    ```
* 当`fstream`对象被销毁，`close()`会被自动调用。

## 文件模式

对于`ofstream`，

* 未指定打开模式时，以`out`打开；
* 通常`out`意味着同时使用`trunc`；
* `app`不能与`trunc`同时设置；
* `app`和`ate`；
    * `app`，所有的输出操作都在文件末尾，不能seek around；
    * `ate`，初始位置在文件末尾，可以seed around。

# `sstream`

```flow
op1=>operation: iostream
op2=>operation: sstream

op2(right)->op1
```

类似`fstream`，`sstream`也包含特有的操作，

* `sstream::sstream(s)`：创建对象，并保存`s`的copy；
* `sstream::str()`：返回保存的string的copy；
* `sstream::str(s)`：copy`s`到对象中。
