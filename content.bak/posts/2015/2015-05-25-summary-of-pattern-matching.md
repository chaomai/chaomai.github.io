title: "Pattern matching相关算法小结"
date: 2015-05-25 11:33:43
description: Pattern matching的算法有很多，这里做一个简单的总结。
categories:
    - algorithms
tags:
    - 'pattern matching'
---

# Pattern matching

Pattern matching的算法有很多，这里做一个简单的总结。

问题：给定一个字符串`txt[0...n-1]`和另一个字符串`pat[0...m-1]`，假设n > m，实现一个函数`search(char pat[], char txt[])`，完成在`txt`中找到所有`pat`出现的位置。

例子：

* Input:

```
txt[] = "THIS IS A TEST TEXT"
pat[] = "TEST"
```

Output:

```
Pattern found at index 10
```

* Input:

```
txt[] = "AABAACAADAABAAABAA"
pat[] = "AABA"
```

Output:

```
Pattern found at index 0
Pattern found at index 9
Pattern found at index 13
```

# Naive Pattern Searching

也叫Bruce Force。

方法很简单，对于`txt`中的每个index i，检查`pat`的每个字符是否匹配。如果有匹配不上的字符，或者匹配成功，都move到下一个index。

最坏时间复杂度，$O(mn)$。

## References

* [Naive Pattern Searching](http://www.geeksforgeeks.org/searching-for-patterns-set-1-naive-pattern-searching/)

# A Better Naive Pattern Searching

这个方法需要有一个前提条件：`pat`里所有字符都不相同。

在这个前提下，如果在匹配了j个字符之后出现了mismatch，那么`pat`就不是后移一个位置，而是后移j个位置。当然如果是首字符就不匹配，那么仍然是后移一个位置。

## References

* [A Naive Pattern Searching Question](http://www.geeksforgeeks.org/pattern-searching-set-4-a-naive-string-matching-algo-question/)

# KMP Algorithm

Naive Pattern Searching在这样的情况下，效率是很低的：

```
txt[] = "AAAAAAAAAAAAAAAAAB"
pat[] = "AAAAB"
```

因为`pat`每次在比较到不同的字符B的时候，仅仅向后移动一位，搜索位置又要退回，重新比较已经比较过的字符。

同时这也是Naive Pattern Searching的改进方法无法处理的，因为里面出现了重复的字符。如果再回去看Naive Pattern Searching的改进方法，其实本质上就是为了避免搜索位置的退回。

KMP可以利用已经比较过的字符这一信息来避免搜索位置退回。至于怎么利用，就是部分匹配表。

部分匹配表是KMP的关键，生成的方式是对pat的每个前缀，计算该前缀的前缀和后缀的最长的共有元素的长度。例如：

```
pat                 A B C D A B D
partial match value 0 0 0 0 1 2 0
```

有了部分匹配表，在发现不同的字符的时候，就不直接把`pat`后移一位，而是根据下面的公式，

$$
移动位数 = 已匹配的字符数 - 对应的部分匹配值
$$

这里对应的部分匹配值指的是，在`pat`中，最后已匹配字符对应的部分匹配值。

如果是首字符就不匹配，那么仍然是后移一个位置。

"部分匹配"的实质是，有时候，字符串头部和尾部会有重复。比如，`"ABCDAB"`之中有两个`"AB"`，那么它的"部分匹配值"就是2（"AB"的长度）。搜索词移动的时候，第一个`"AB"`向后移动4位（$字符串长度-部分匹配值$），就可以来到第二个`"AB"`的位置。

最坏时间复杂度，$O(n)$。

看完KMP，可以发现Naive Pattern Searching的改进方法实际上是KMP的特例，由于pat中所有字符都不相同，因此部分匹配表中所有的对应的部分匹配值都是0，

$$
移动位数 = 已匹配的字符数
$$

## References

* [KMP Algorithm](http://www.geeksforgeeks.org/searching-for-patterns-set-2-kmp-algorithm/)
* [字符串匹配的KMP算法](http://www.ruanyifeng.com/blog/2013/05/Knuth%E2%80%93Morris%E2%80%93Pratt_algorithm.html)

# Rabin-Karp Algorithm

Bruce Force在`txt`上每次把`pat`后移一位，每次移动之后，检查`pat`的每个字符是否匹配。Rabin-Karp也是类似的，每次把pat后移一位，不同的是Rabin-Karp比较的是`pat`的hash值和当前对应的`txt`子串的hash值。如果hash值相等，然后再去逐个检查子串的字符。

最直接的方法莫过于计算`h(pat)`和`txt`中所有子串的hash，然后一一比较。但光是计算`txt`中所有子串的hash就需要O(mn)的时间，这样一来，相比起Naive Pattern Searching，这个方法就毫无优势了。

如何计算hash值是Rabin-Karp的关键，最好是能够利用当前`txt`子串的hash值，计算后移一位以后的，以减少计算的开销。Rabin-Karp使用的hash叫做Rolling hash，基本实现是刚刚的方法实际上重复计算了很多重叠的部分，而Rolling hash就要利用当前子串的hash值，来计算后移一个位置之后子串的hash值。

Intro to Algorithms的Lecture Note举了一个很形象的例子来说明Rolling hash。

在最坏情况下，每次移动后hash值都相等（因为子串相同或hash冲突），因此移动后都要逐个检查子串的字符。

最坏时间复杂度，$O(mn)$。

## References

* [Rolling hash](http://en.wikipedia.org/wiki/Rolling_hash)
* [Rolling Hash (Rabin-Karp Algorithm)](http://courses.csail.mit.edu/6.006/spring11/rec/rec06.pdf)
* [Hashing-II](http://stellar.mit.edu/S/course/6/fa13/6.006/courseMaterial/topics/topic6/lectureNotes/L09-Hashing-II/L09-Hashing-II.pdf)
* [Rabin-Karp Algorithm](http://www.geeksforgeeks.org/searching-for-patterns-set-3-rabin-karp-algorithm/)

# Finite Automata

基于有限状态机实现pattern searching，就是用pattern来构建一个状态表，构建完成以后就可以根据txt的每个字符，来在有限状态机的各个状态之间转移，如果到达终态，那就是匹配到了。

这个算法的关键就是基于pattern构建状态表。

## References

http://www.geeksforgeeks.org/searching-for-patterns-set-5-finite-automata/
