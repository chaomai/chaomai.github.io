title: 从汇编的角度分析C++引用
date: 2015-03-10 17:35:29
description: C++中，引用为对象起了另外一个名字，引用类型refers to另外一种类型。引用和指针是不同的，可以汇编的角度来看引用。
categories:
  - cpp
tags:
  - 'cpp reference'
---

# 引用

C++中，引用为对象起了另外一个名字，引用类型refers to另外一种类型。定义引用时，把引用和初始值绑定在一起，而不是将初始值拷贝给引用。定义了引用之后，对其进行的所有操作都是在与之绑定的对象上进行的。但是引用本身不是一个对象。

说到这里，其实会发现引用和指针有点像，但实际上它们是不同的。首先引用在绑定到对象以后，就不能再绑定到另外一个对象；其次，引用本身不是一个对象，但是指针是一个对象。

# 汇编的角度

更深入到底层，可以汇编的角度来看引用。

```cpp
int a = 3;
int &ra = a;
ra++;

int b = 4;
int *pa = &b;
pa++;
(*pa)++;
```

以上代码的反汇编如下：

```
int a = 3;
012E33F8  mov         dword ptr [a],3
int &ra = a;
012E33FF  lea         eax,[a]
012E3402  mov         dword ptr [ra],eax

int b = 4;
012E3405  mov         dword ptr [b],4
int *pa = &b;
012E340C  lea         eax,[b]
012E340F  mov         dword ptr [pa],eax


ra++;
013F4475  mov         eax,dword ptr [ra]
013F4478  mov         ecx,dword ptr [eax]
013F447A  add         ecx,1
013F447D  mov         edx,dword ptr [ra]
013F4480  mov         dword ptr [edx],ecx

pa++;
013F448F  mov         eax,dword ptr [pa]
013F4492  add         eax,4
013F4495  mov         dword ptr [pa],eax
(*pa)++;
013F4498  mov         eax,dword ptr [pa]
013F449B  mov         ecx,dword ptr [eax]
013F449D  add         ecx,1
013F44A0  mov         edx,dword ptr [pa]
013F44A3  mov         dword ptr [edx],ecx
```

可以看到，首先把3放入地址为[a]的内存，然后把a的地址放入eax，最后把eax的值放入地址为[ra]的内存。实际上，就是把a的地址放入了ra里。而b和pa也同样是这样步骤。

然后再来看++操作的部分，在汇编的角度，引用和指针在内存中都是地址，在对指针指向的变量进行++时，需要手动的来进行解引用；但对于引用，解引用这个操作是编译器帮你完成了，只需要直接++即可。

从汇编的角度来看，引用是通过指针来实现的。

# 实现引用类型与被引用对象分离?

C++中规定了引用在绑定到对象以后，就不能再绑定到另外一个对象，既然了解了C++中引用的底层的实现，能否通过底层的方法来绕过这个限制？答案是可以的。

先来看这么几行代码，

```cpp
int a = 3;
int b = 4;
int &ra = a;
```

对ra进行的所有操作都是在与之绑定的变量a上进行的，因此直接操作ra来修改绑定是无法实现的。由于定义以上几个变量时，它们应该是处于相邻的内存空间中，因此可以通过ra相邻的内存，来更改ra，进而分离ra与被引用对象a。

这是在执行以上3条语句之后的内存情况，

```
0x0078F750  cc cc cc cc 6c f7 78 00  ????l?x.
0x0078F758  cc cc cc cc cc cc cc cc  ????????
0x0078F760  04 00 00 00 cc cc cc cc  ....????
0x0078F768  cc cc cc cc 03 00 00 00  ????....
0x0078F770  cc cc cc cc 96 d1 e4 4e  ???????N
0x0078F778  c8 f7 78 00 69 69 18 00  ??x.ii..
```

其中EBP=其中EBP=0x007EFA7C，可以看到EBP之前的12byte的位置才是第一个变量，再往前12byte是第二个变量，继续往前12byte是引用。因此b的地址减3就是存储引用的内存，修改这个地方即可。

```cpp
*(&b - 3) = (int)&b;
ra++;
```

继续执行下面代码以后，b被增加成5。

# 参考

1. C++ Primer 第5版
