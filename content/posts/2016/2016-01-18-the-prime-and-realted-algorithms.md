---
title: 素数相关问题及算法
date: 2016-01-18 15:38:59
categories:
    - algorithms
tags:
    - prime
---

# 试除法

根据素数的定义，对于自然数$n$，只要能够找到除了1和它本身以外，能够整除该数的正整数，那么它就不是素数。

```cpp
bool isPrime(int n) {
    if (n <= 1) {
        return false;
    }

    for (int i = 2; i <= n; ++i) {
        if (n % i == 0 && i != n) {
            return false;
        }
    }

    return true;
}
```

又因为，如果$d$是$n$的约数，那么$n=d \times n/d$，故$n/d$也是，且$\min{d, n/d} \leq \sqrt{n}$,

因此只需要测试到$\sqrt{n}$即可。

```cpp
bool isPrime(int n) {
    if (n <= 1) {
        return false;
    }

    for (int i = 2; i * i <= n; ++i) {
        // i won't equal to n, so i != n is unnecessary.
        if (n % i == 0) {
            return false;
        }
    }

    return true;
}
```

上述方法需要约$2^{n/2}/((n/2)\ln 2)$次测试。

# 埃氏筛

对于单个小整数，试除法尚可，但是如果要枚举$n$以内的素数，使用试除法就需要对$n$以内的所有自然数都做测试，效率太低。

埃拉托斯特尼筛法，简称埃氏筛，从2开始，将每个素数的所有倍数标记为合数，如果这个$n$小于最后一个标出的素数的平方，那么所有小于$n$且未标记的都是素数。

```cpp
vector<bool> isPrime(n, true);
isPrime[0] = isPrime[1] = false;

int i = 0;

while (i * i < n) {
    if (isPrime[i]) {
        int j = 2;
        while (i * j <= n) {
            isPrime[i * j] = false;
            ++j;
        }
    }

    ++i;
}
```

埃氏筛的时间复杂度为$O(n\log n \log n)$，但这个方法最大的问题在于，需要的内存随着$n$的增大而增大。

# 整数分解

## 试除法

分解整数，普通方法是用短除法，从最小的素数除起，知道结果为素数为止。

```cpp
for (int i = 2; i * i <= n; ++i) {
    while (n % i == 0) {
        std::cout << i;
        n /= i;
    }
}

if (n > 1) {
    std::cout << n;
}
```

既然是从最小的素数除起，上述代码在每次`++i`后，为什么不检测`i`是否为素数？

原因是，内层循环相当于已经把当前`i`的所有倍数去除了，类似埃氏筛，故下次取到的能够整除$n$的`i`是素数。

# References

1. [Trial division](https://en.wikipedia.org/wiki/Trial_division)
2. [Sieve of Eratosthenes](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes)
3. [Prime](http://algorithm.yuanbin.me/zh-hans/basics_algorithm/math/prime.html)
