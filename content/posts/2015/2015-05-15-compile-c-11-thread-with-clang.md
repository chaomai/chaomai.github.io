title: Clang++中C++11 thread的支持
date: 2015-05-15 19:35:23
description: 在Linux下编译C++11的多线程代码时，提示pthread_create为定义。最后发现clang++在Linux在使用的是posix来实现C++11的线程，编译的时候需要加上-pthread。
categories:
    - cpp
tags:
    - clang
    - thread
---

# 问题
在Ubuntu中使用Clang++，

```bash
clang++-3.7 -std=c++11 test.cpp
```

编译C++11编写的多线程代码时，

```cpp
#include <iostream>
#include <thread>

void f()
{
  std::cout << "Hello World\n";
}

int main()
{
  std::thread t(f);
  t.join();
}
```

发现如下问题：

```bash
/tmp/test-9606ba.o: In function `std::thread::thread<void (&)()>(void (&)())':
test.cpp:(.text[_ZNSt6threadC2IRFvvEJEEEOT_DpOT0_]+0x21): undefined reference to `pthread_create'
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

# 分析

1. 在Linux中，Standard C++ library的默认实现是libstdc++。虽然安装了clang，但是编译时使用的仍然是GNU的libstdc++。

2. 执行clang++ -v以后，可以知道使用的线程模型是posix。

# 解决

既然底层使用了posix来实现C++11的线程，那么编译的时候必然要有-pthread

```bash
clang++-3.7 -std=c++11 -pthread test.cpp
```
