---
title: scanRight和动态规划
date: 2017-04-23 14:16:51
categories:
    - "functional programming"
tags:
    - scanRight
---

# scanRight

这是 Functional Programming in Scala 一书中，练习5.16的一个问题。问题是，把 `tail` 泛化为 `scanRight`，这个 `scanRight` 需要返回中间结果形式的 stream。例如：

```scala
scala> Stream(3, 2, 1).scanRight(0)(_ + _).toList
res0: List[Int] = List(6, 5, 3, 0)
```

*注：1. 给定一个 stream，`tail` 会将这个输入的所有后缀作为 stream 返回；2. 中间结果形式的 stream 意味着结果是以 call-by-name 的方式构造的；3. 遍历 `scanRight` 的时间复杂度应该为 $O(n)$。*

先不考虑时间复杂度的要求，仅仅实现 `scanRight` 的功能，那很容易就可以写出下面的代码，

```scala
def scanRight1[B](z: B)(f: (A, => B) => B): Stream[B] =
  foldRight((z, Stream(z)))((e, acc_s) => {
    val acc_e = f(e, acc_s._1)
    (acc_e, Stream.cons(acc_e, acc_s._2))
  })._2
```

`foldRight` 保留了累计值 `acc_e` 和 stream 的中间值 `acc_s`，在每次迭代（这里实际上是递归）中，使用这两个值来进行 `Stream.cons`，以得到最后结果。


# 计算过程分析

在看这题的答案的时候，发现答案并没有直接使用 `acc_s`，而是用 `lazy` 缓存了这个值，然后基于这个缓存的值进行计算。

```scala
def scanRight2[B](z: B)(f: (A, => B) => B): Stream[B] =
  foldRight((z, Stream(z)))((e, acc_s) => {
    lazy val p = acc_s
    val acc_e = f(e, p._1)
    (acc_e, Stream.cons(acc_e, p._2))
  })._2
```

这里我不理解的是缓存什么不好，为什么要缓存 `acc_s`。因为每次迭代都会遇到 stream 中的元素，为什么不 `lazy val p = e`。

下面以 `Stream(3, 2, 1).scanRight(0)(_ + _)` 为例，分析 `scanRight1` 计算过程。

1. 把求和的 function 叫 sum，

    ```scala
    s.scanRight(0)(sum)
    ```

2. 展开 scanRight，

    ```scala
    s.foldRight((0, Stream(0)))((e, acc_s) => {
      val acc_e = sum(e, acc_s._1)
      (acc_e, Stream.cons(acc_e, acc_s._2))
    })._2
    ```

3. 为了后续分析方便，对比 `foldRight` 的类型，提取出 `foldRight` 的 `f`，

    ```scala
    def foldRight[B](z: => B)(f: (A, => B) => B): B

    f: (Int, (Int, Stream[Int])) => (Int, Stream[Int]) = (e, acc_s) => {
      val acc_e = sum(e, acc_s._1)
      (acc_e, Stream.cons(acc_e, acc_s._2))
    }
    ```

4. 此时可以进行 `foldRight` 中的 pattern matching 了，

    ```scala
    Cons(h, t) =>
    h: 3
    t: Stream(2, 1).foldRight((0, Stream(0)))(f)
    ```

5. 把参数代入 `f` 进行计算，

    ```scala
    acc_e: 3
    acc_s: Stream(2, 1).foldRight((0, Stream(0)))(f)

    {
      val acc_e = sum(3, acc_s._1)
      (acc_e, Stream.cons(acc_e, acc_s._2))
    }
    ```

    根据 `foldRight` 的定义，逐层展开 `acc_s._1`，即：`Stream(2, 1).foldRight((0, Stream(0)))(f)._1`，省略了展开的详细过程和 `foldRight` 中对 `Empty` 的 pattern matching，

    ```scala
    {
      val acc_e = sum(3,
                        {
                          val acc_e = sum(2,
                                            {
                                              val acc_e = sum(1, 0)

                                              (acc_e, Stream.cons(acc_e, acc_s._2))
                                            }._1
                                          )

                          (acc_e, Stream.cons(acc_e, acc_s._2))
                        }._1
                      )

      (acc_e, Stream.cons(acc_e, acc_s._2))
    }
    ```

    等价于，

    ```scala
    {
      val acc_e = 6
      (6, Stream.cons(6, acc_s._2))
    }
    ```

6. 接下来计算 `acc_s._2`，即：`Stream(2, 1).foldRight((0, Stream(0)))(f)._2`。

观察上述过程发现，`Stream(2, 1).foldRight((0, Stream(0)))(f)`，也就是 `acc_s` 会被计算两次，这就是 `lazy val p = acc_s` 的缘由。


# 与动态规划的联系

进一步分析，在第一次迭代中，当前的 `acc_s` 需要计算两次。在后续的迭代中（计算：`(Stream.cons(6, acc_s._2)`），每次迭代都重复计算了一部分结果。简单来说就是，

```
第一次：3 + 2 + 1
第二次：    2 + 1
第三次：        1
```

例如值 `1`，在 3 次迭代中，都被重复计算得到了 3 次。使用 `lazy val p = acc_s` 避免了这些多余的计算。

这时回过头去看 `scanRight`，可以将这个函数的功能看做是，对所有的后缀 stream 进行了 `foldRight` 操作。对长后缀 stream 计算 `foldRight` 时，实际上计算了短后缀 stream 的 `foldRight`。这实际上就是动态规划，

* 最优子结构：当前长后缀 stream 的 `foldRight`可以由短后缀 stream 得到。
* 无后效性：短后缀 stream 的 `foldRight` 不受长后缀 stream 的影响。
* 子问题重叠：短后缀 stream 的 `foldRight` 会被多次计算。

由于子问题重叠，因此常用，

* Memoization (Top Down)
* Tabulation (Bottom Up)

来存储子问题的解，避免重复计算。

`scanRight` 实际上就是用了 Memoization 来存储子问题的解。
