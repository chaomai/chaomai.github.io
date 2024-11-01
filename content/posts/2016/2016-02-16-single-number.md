title: Single Number
date: 2016-02-16 19:59:44
categories:
    - algorithms
tags:
    - 'bit manipulation'
---

# Single Number I

问题：给定一个数组，除了其中一个数出现一次，其他数都出现了两次，找到这个数。

最简单的办法用莫过于用map统计每个数出现的次数，但这除了需要遍历数组外，还需要遍历map，且需要O(n)空间。

使用异或操作更为方便，根据`a xor a = 0`和`a xor 0 = a`，

```cpp
int singleNumber(vector<int>& nums) {
  int ret = 0;

  for (auto n : nums) {
    ret ^= n;
  }

  return ret;
}
```

成对的数字进行xor后是0，最后ret便是只出现一次的那个数字。

# Single Number II

## 方法1

问题：给定一个数组，除了其中一个数出现一次，其他数都出现了三次，找到这个数。

此时I的xor方法不可用，因为出现次数是奇数次。

考虑每个数的二进制表示，但凡这个数出现了三次，那么二进制表示中，如果某个bit有1，则这个1也在该bit上出现了三次。统计所有数字每个bit上1出现的次数，最后mod 3，余数就是那个数相应bit上的值。

```
2    2    2    11   11   11   12
0010 0010 0010 1011 1011 1011 1000
                              ----
count 1 for each bit,
bit index: 3 2 1 0
count:     4 0 6 3
mod 3:     1 0 0 0
```

以下是代码，

```cpp
int singleNumber(vector<int>& nums) {
  vector<int> count(32, 0);

  for (auto n : nums) {
    for (int j = 0; j != count.size(); ++j) {
      if ((n >> j) & 1 != 0) {
        count[31 - j] += 1;
      }
    }
  }

  int result = 0;
  for (int i = 0; i != count.size(); ++i) {
    result |= (count[31 - i] % 3) << i;
  }

  return result;
}
```

以上代码是遍历每个数，统计每个数中bit为1的数目。还可以遍历每个bit，统计所有数字该bit上1的数目，这样可以合并两个for。

```cpp
int singleNumber(vector<int>& nums) {
  vector<int> count(32, 0);
  int result = 0;

  for (int i = 0; i != count.size(); ++i) {
    for (auto n : nums) {
      count[31 - i] += (n >> i) & 1;
    }
    result |= (count[31 - i] % 3) << i;
  }
  return result;
}
```

## 方法2

但是上述方法需要O(n)的空间。在这个[discuss](https://leetcode.com/discuss/857/constant-space-solution)中看到了更好的方法，结合例子能明白做了什么，但是无法理解这样位操作的idea是如何想出来的。。。最后在另一个[blog]((http://traceformula.blogspot.com/2015/08/single-number-ii-how-to-come-up-with.html))和另一个[disscus](https://leetcode.com/discuss/81165/explanation-manipulation-method-might-easier-understand)中找到了另一个方法，用了卡诺图。

在上面的代码中，每个bit的计数器`count[31 - i]`值为多少是无所谓的，最终需要的只是`count[31 - i] % 3`的结果。每当计数器为3，可以把它重置为0，即0 (00) -> 1 (01) -> 2 (10) -> 0 (00)，因此可以使用一个Two Bit Counter来表示状态的转换，

```
对于数字的每个bit b，
old_two, old_one b -> two, one
0        0       0    0    0
0        1       0    0    1
1        0       0    1    0
0        0       1    0    1
0        1       1    1    0
1        0       1    0    0
```

上图变为卡诺图，

```
two:                       one:
     old_two old_one       old_two old_one
     00 01 10              00 01 10
b 0   0  0  1         b 0   0  1  0
  1   0  1  0           1   1  0  0
```

可以得到，

```
two = (old_two & ~old_one & ~b) | (~old_two & old_one & b)
one = (~old_two & old_one & ~b) | (~old_two & ~old_one & b)
```

由上面的式子可得，

```cpp
int singleNumber(vector<int>& nums) {
  vector<int> count(32, 0);
  int result = 0;

  for (int i = 0; i != count.size(); ++i) {
    int two = 0;
    int one = 0;

    int old_two, old_one;
    for (auto n : nums) {
      old_two = two;
      old_one = one;
      int b = (n >> i) & 1;

      two = (old_two & ~old_one & ~b) | (~old_two & old_one & b);
      one = (~old_two & old_one & ~b) | (~old_two & ~old_one & b);
    }

    result |= (one << i);
  }

  return result;
}
```

接着发现blog里面实际上是对每个数字使用了上面的公式，而不是对数字的每个bit。观察上面的状态转换图和代码，对每个bit进行计算的时候，two和one始终都是在使用最后一个bit，没有进位的情况，如果多个bit同时计算，没有相互影响，不会有问题，

```cpp
int singleNumber(vector<int>& nums) {
  int two = 0;
  int one = 0;

  for (auto n : nums) {
    int old_two = two;
    int old_one = one;

    two = (old_two & ~old_one & ~n) | (~old_two & old_one & n);
    one = (~old_two & old_one & ~n) | (~old_two & ~old_one & n);
  }

  return one;
}
```

上面的公式还可简化，计算two时，由于`old_two = two`且`old_one = one`，

```
two = (two & ~one & ~b) | (~two & one & b)
    = one & (two ^ b)
one = (~old_two & one & ~b) | (~old_two & ~one & b)
```

对one还可以简化，在状态转换表中用11代替old_two和old_one的10，进而在式子中消除old_two（`one = two | (one ^ b)`）。还有，如果使用了不同的状态转换表示，最后的结果也会有所差异。

# Single Number III

问题：给定一个数组，除了其中两个数分别出现一次，其他数都出现了两次，找到这两个数。

用I的方法来做的话，ret是那两个数的xor，因此现在需要从ret中区分出两个数。

假设有如下数组，且ret中只有第3个bit被置为1，

```
[a, b, c, b, d, a]
ret = c ^ d
```

如果ret中某一bit出现了1，那么这两个数中，对应的bit必然是一个为1，另一个为0。根据该bit是否为1，可以把原数组分为两组，c和d不会在同一个组里，有下面两种情况，

```
1. 某个组里只有c或d
[c]
[a, b, d, b, a]
c = c
a ^ b ^ d ^ b ^ a = d

[a b c b a]
[d]
a ^ b ^ c ^ b ^ a = a
d = d

2. 每个组除了c或d，还有其他数
[a c a]
[b d b]
a ^ c ^ a = c
b ^ d ^ b = d
```

无论是情况1或2，如果某个组除了c或d，还有其他数，那么这些数必定是成对出现的。接下来把两个组的数分别xor即可，两个xor的结果就是c和d。

上述例子是假设了ret中只有一个bit被置为1，如果ret中有多个bit为1，则只需要根据其中的一个bit来分组，因此需要从ret中提取出一个为1的bit。下面提取最后一个为1的bit，

```
last_bit_one = r & ~(r - 1);

若ret为1010010，
ret           1010010
r- 1          1010001
~(r - 1)      0101110
r & ~(r - 1)  0000010
```

```cpp
vector<int> singleNumber(vector<int>& nums) {
  int r = 0;

  for (auto n : nums) {
    r ^= n;
  }
  int last_bit_one = r & ~(r - 1);

  vector<int> res(2, 0);
  for (auto n : nums) {
    if ((last_bit_one & n) != 0) {
      res[0] ^= n;
    } else {
      res[1] ^= n;
    }
  }

  return res;
}
```

# 扩展

## I

问题：给定一个数组，除了其中一个数出现两次，其他数都出现了三次，找到这个数。

类似Single Number II，如果用vector对值为1的bit进行计数，那么最后mod 3，如果余数为2，那么该bit就是那个数字的相应bit，代码改为`result |= (count[31 - i] % 3 == 2) << i;`。

如果是使用第二个方法，直接返回two即可。

## II

问题：给定一个数组，除了其中一个数出现l次，其他数都出现了k次，找到这个数。

### References

1. [My own explanation of bit manipulation method, might be easier to understand](https://leetcode.com/discuss/81165/explanation-manipulation-method-might-easier-understand)
2. [Constant space solution](https://leetcode.com/discuss/857/constant-space-solution)
3. [leetcode Single Number II - 位运算处理数组中的数](http://www.cnblogs.com/daijinqiao/p/3352893.html)
4. [Explain using xor to find two non-duplicate integers in an array](http://stackoverflow.com/questions/22952651/explain-using-xor-to-find-two-non-duplicate-integers-in-an-array)
5. [Single Number II : How to come up with Bit Manipulation Formula](http://traceformula.blogspot.com/2015/08/single-number-ii-how-to-come-up-with.html)
