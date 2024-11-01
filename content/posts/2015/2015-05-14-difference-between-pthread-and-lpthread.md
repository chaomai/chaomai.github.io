title: 编译时，-pthread and -lpthread的区别
date: 2015-05-14 19:31:30
description: gcc编译多线程代码时，参数不同导致结果的不同。
categories:
    - linux
tags:
    - pthread
    - thread
---

-pthread告诉编译器，要链接到pthread库，同时配置线程的编译。

下面的例子就显示了在使用-pthread时，定义了不同的宏。

```bash
$ gcc -pthread -E -dM test.c > dm.pthread.txt
$ gcc          -E -dM test.c > dm.nopthread.txt
$ diff dm.pthread.txt dm.nopthread.txt
152d151
< #define _REENTRANT 1
208d206
< #define __USE_REENTRANT 1
```

-lpthread只会告诉编译器，要链接到pthread库，但是这些宏不会被定义。

编译时，应该使用-pthread。

# reference

1.  [difference-between-pthread-and-lpthread-while-compiling](http://stackoverflow.com/questions/23250863/difference-between-pthread-and-lpthread-while-compiling)
