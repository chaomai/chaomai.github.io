---
title: 函数式编程相关
date: 2017-05-12 09:10:55
categories:
    - "functional programming"
tags:
    - scala
---

Notes of Functional Programming in Scala and more.

## Pure Function

对于一个函数 `f: A => B`，如果任何一个值 `a: A` 都有只有一个 `b: B` 与之对应，使得 `b` 仅仅由 `a` 决定，那么这个函数是纯函数。任何内部或外部状态的改变都不影响 `f(a)` 的结果。


## Referential Transparency

引用透明是表达式的属性。如果一个表达式是引用透明的，那么把这个表达式替换为它求值后的结果不会影响程序的行为。引用透明是定义一个纯函数的必要条件。

如果表达式不是引用透明的，那么这个表达式就是有 side-effect（副作用）。根据引用透明的定义，对于一个纯函数函数 `f`，表达式 `f(x)` 不会有任何的副作用。


## Idempotence（幂等性）

纯函数的幂等的。

$$
\forall x, f(f(x)) = f(x)
$$


# Currying

一个 Curryied 函数是，把一个接受多个参数的函数重写为，一个首先接受第一个参数并返回一个函数接受第二个参数...的函数。

例如：foldLeft。

https://oldfashionedsoftware.com/2009/07/10/scala-code-review-foldleft-and-foldright/


# Currying 和 Partially Applied Function

* http://stackoverflow.com/questions/14309501/scala-currying-vs-partially-applied-functions
* http://stackoverflow.com/questions/8650549/using-partial-functions-in-scala-how-does-it-work/8650639#8650639


# trait and class

* http://stackoverflow.com/questions/1991042/what-is-the-advantage-of-using-abstract-classes-instead-of-traits
* http://stackoverflow.com/questions/2005681/difference-between-abstract-class-and-trait
* http://www.artima.com/pins1ed/traits.html#12.7


# Functions and Methods

In Scala there is a rather arbitrary distinction between functions defined as _methods_, which are introduced with the `def` keyword, and function values, which are the first-class objects we can pass to other functions, put in collections, and so on.

* https://www.google.com/search?q=difference+scala+function+method
* http://www.cnblogs.com/shihuc/p/5082701.html
* http://stackoverflow.com/questions/4839537/functions-vs-methods-in-scala
* http://stackoverflow.com/questions/2529184/difference-between-method-and-function-in-scala
* http://jim-mcbeath.blogspot.com/2009/05/scala-functions-vs-methods.html
* https://dzone.com/articles/revealing-scala-magician%E2%80%99s
* https://tpolecat.github.io/2014/06/09/methods-functions.html
* https://www.quora.com/What-is-the-difference-between-function-and-method-in-Scala
* https://softwarecorner.wordpress.com/2013/09/06/scala-methods-and-functions/
* http://www.marcinkossakowski.com/difference-between-functions-and-methods-in-scala/
* http://japgolly.blogspot.com/2013/10/scala-methods-vs-functions.html
* http://blog.vmoroz.com/posts/2016-06-01-scala-should-i-use-a-method-or-a-function.html


# Functional Combinators

* [Explanation of combinators](http://stackoverflow.com/questions/7533837/explanation-of-combinators-for-the-working-man)

`List(1, 2, 3) map squared`把函数`squared`应用到了`List(1, 2, 3)`的每个元素上，并返回一个新的List。这个操作叫做map组合子（map combinators）。


# PartialFunction and Partially Applied Function


# Rank-1 polymorphism and Higher-Rank Polymorphism

* https://apocalisp.wordpress.com/2010/07/02/higher-rank-polymorphism-in-scala/
    * related:
    * https://apocalisp.wordpress.com/2010/10/26/type-level-programming-in-scala-part-7-natural-transformation%C2%A0literals/
    * https://apocalisp.wordpress.com/2011/03/20/towards-an-effect-system-in-scala-part-1/
* http://existentialtype.net/2008/03/09/higher-rank-impredicative-polymorphism-in-scala/


# Existential Types

* http://www.drmaciver.com/2008/03/existential-types-in-scala/


# Algebraic Data Type

* http://tpolecat.github.io/presentations/algebraic_types.html
* https://www.youtube.com/watch?v=YScIPA8RbVE


# Variant

## Covariant and contravariant

对于所有类型`X`和`Y`，以及类型`Co[+A]`，`Ctr[-A]`，`Iv[A]`。如果X是Y的子类，

covariant：`Co[X]`是`Co[Y]`的子类。在类型参数A前加上+表示 covariant。
contravariant：`Ctr[Y]`是`Ctr[X]`的子类。在类型参数A前加上-表示 contravariant。
invariant：`Iv[X]`和`Iv[Y]`无关。

```
┌───┐     ┌───────┐     ┌───────┐
│ Y │     │ Co[Y] │     │Ctr[X] │
└───┘     └───────┘     └───────┘
  ▲           ▲             ▲
  │           │             │
  │           │             │
┌───┐     ┌───────┐     ┌───────┐
│ X │     │ Co[X] │     │Ctr[Y] │
└───┘     └───────┘     └───────┘
```

表述 covariant 的另一种方式是，在所有上下文中，都可安全的把`A`转换为`A`父类。contravariant 类似。

```scala
class Animal {
  val sound = "rustle"
}

class Bird extends Animal {
  override val sound = "call"
}

class Chicken extends Bird {
  override val sound = "cluck"
}

class covariant1[+A]

// error:
// val covariant_b1: covariant1[Bird] = new covariant1[Animal]
val variant_b2: variant1[Bird] = new variant1[Chicken]
                │            │       │               │
                └────────────┘       └───────────────┘
                      ▲                       │
                      └───────────────────────┘
                           to supper class

class contravariant2[-A]

val contravariant_b3: contravariant2[Bird] = new contravariant2[Animal]
// error:
// val contravariant_b4: contravariant2[Bird] = new contravariant2[Chicken]
```


## Covariant and Contravariant Positions

```scala
trait Function1 [-T1, +R] extends AnyRef
```

以函数为例，

contravariant (positive) position：一个类型在函数参数的类型中。更一般的来说，是函数所使用的值的类型。
covariant (negative) position：一个类型在函数结果的类型中。更一般的来说，是函数产生的值的类型。

```
                   ┌────────────────────────┐
                   │ contravariant position │
                   └────────────────────────┘
                                ▲
             ┌──────────────────┘
             │
  def foo(a: A): B
                 │
                 └────────────┐
                              ▼
                   ┌────────────────────┐
                   │ covariant position │
                   └────────────────────┘
```

对于高阶函数，从外层向内分析。类型最终是什么 position 由分析过程中，各个类型的“累加”得到。就函数 `foo` 而言，`A => B` 是在 contravariant position；就函数 `f` 而言， `A` 是在 contravariant position，`B` 是在 covariant position。因此，最终 `A` 是在 covariant position（可以理解为两个 negative position 合并，负负得正），而 `B` 是在 contravariant position。

```
                    ┌───────────────────────┐
                    │ A: covariant position │
                    └───────────────────────┘
                                ▲
                                │                    ┌───────────────────────────┐
                                                     │ B: contravariant position │
                ┌ ─ ─ ─ ─ ─ ─ ─ ┴ ─ ─ ─ ─ ─ ─ ─ ─    └───────────────────────────┘
                                                 │                 ▲
                ├──────────────────────────────────────────────────┴──────────┐
                │                                │                            │
                │                   ┌────────────────────────┐                │
   ┌────────────────────────┐       │ contravariant position │     ┌────────────────────┐
   │ contravariant position │       └────────────────────────┘     │ covariant position │
   └────────────────────────┘                    ▲                 └────────────────────┘
                ▲                    ┌───────────┘                            ▲
                │                    │    ┌───────────────────────────────────┘
                │                    │    │
             ┌────┐       ┌───────▶  A => B
             │    │───────┘
  def foo(f: A => B): C
                      │
                      └────────────┐
                                   ▼
                        ┌────────────────────┐
                        │ covariant position │
                        └────────────────────┘
```

Scala要求类型本身的variant与类型所在位置的variant一致。

```scala
// error 1
trait Option[+A] {
  def orElse(o: Option[A]): Option[A]
  ...
}

// error 2
class V1[+A]
class V2[-B] extends V1[B]
```

这个定义将会导致编译错误，`Error:... covariant type A occurs in contravariant position in type`。`orElse` 函数接受参数`Option[A]`，这个位置（contravariant position）是只能将 `A` 转换为 `A` 的子类型的地方。但是类型 `A` 是 covariant 的，也就是说在所有上下文中，都可安全的把 `A` 转换为 `A` 父类。这里出现了冲突。

对于 `orElse`，解决方式是不使用 `A`，使用边界限定类型为 `A` 的父类。

```scala
def orElse[B >: A](o: Option[B]): Option[B] = this match {
    case None => ob
    case _ => this
}
```

那为什么不，

```scala
// 1.
def orElse[B <: A](o: Option[B]): Option[B]

// 2.
def orElse[B](o: Option[B]): Option[B]
```

`orElse` 返回值可能的类型为 `Option[B]` 或 `Option[A]`，
* 对于1，`B` 是 `A` 的子类，`Option[B]` 是 `Option[A]` 的子类，当返回 `this` 时，类型为 `Option[A]`，这里无法从父类转换到子类。
* 对于2，当返回 `this` 时，类型为 `Option[A]`，和 `Option[B]` 完全无关，类型不匹配。


# Laziness 和 Non-strictness

二者都与求值策略[^1]（evaluation strategy）有关。一个编程语言用求值策略来，

* 确定合适计算一个函数调用的参数。
* 以什么样的形式来把数值传递给函数。

[^1]: https://en.wikipedia.org/wiki/Evaluation_strategy

求值策略包括，

* 严格求值（strict evaluation）：一个函数调用的参数总是在应用这个函数之前进行完全求值。常见的有，
    * call by value
    * call by reference

    关于这个话题，不得不提的是计算顺序[^2]（order of evaluation），在c++中，这个顺序未定义，由编译器决定。
* 非严格求值（non-strict evaluation）：一个函数调用的参数只有在函数体内被使用到时，才进行求值。常见有，
    * call by name：在函数体内被使用到时才进行求值，没有被用到则不会求值。如果被多次使用到，那么每次使用都需要重新求值。
    * call by need：记忆版的 call by name。当参数首次被求值时，值被保留用于后续使用。
* 非确定性策略（nondeterministic strategies）
    * call by future

在 Scala 中，非严格参数又叫做 call "by name" 参数，对应 Scala 中对这些参数使用的 call by name求值策略。当首次使用到某个参数的时候，通过 `lazy val` 缓存这个值，来达到 call by need（又叫做 lazy evaluation）。

[^2]: http://en.cppreference.com/w/cpp/language/eval_order

# Nothing 和 NotNothing

* [Advanced Type Constraints with Type Classes](https://github.com/OlegIlyenko/hacking-scala-blog/blob/master/posts/Advanced-Type-Constraints-with-Type-Classes.md)
* [NotNothing.scala](https://github.com/mongodb/mongo-spark/blob/master/src/main/scala/com/mongodb/spark/NotNothing.scala)
* [Possible to make scala require a non-Nothing generic method parameter and return to type-safety](http://stackoverflow.com/questions/41403287/possible-to-make-scala-require-a-non-nothing-generic-method-parameter-and-return)

Scala 中，关于多态，一个头疼的[问题](https://issues.scala-lang.org/browse/SI-2609)是，如果没有为一个多态方法指定类型参数，并且无法从方法的参数中推断出类型，由于 `Nothing` 是所有类型的子类型，此时 Scala 就会把没有指定的类型推断为 `Nothing`。

```scala
sealed trait Dimension

sealed trait UnitDim extends Dimension

trait VarDim extends Dimension

sealed trait DimValue[D <: Dimension] {
  def value: Int
}

object DimValue {
  def apply[D <: Dimension](v: Int) = new DimValue[D] {
    override def value: Int = v
  }
}
```

上面的 `DimValue`，我希望 `apply` 中，`D` 为 `Dimension` 的子类型。如果在调用这个方法时为明确指定类型参数，结果就会是 `DimValue[Nothing]`。这不是我所期望的。

```scala
val dv = DimValue(3)

Pattern: dv: DimValue[Nothing] with Object {
  def value: Int
}
```

Google 了 Scala NotNothing 以后，找到两个方法，

* [Possible to make scala require a non-Nothing generic method parameter and return to type-safety Ask Question](http://stackoverflow.com/questions/41403287/possible-to-make-scala-require-a-non-nothing-generic-method-parameter-and-return)
* [Advanced Type Constraints with Type Classes](https://github.com/OlegIlyenko/hacking-scala-blog/blob/master/posts/Advanced-Type-Constraints-with-Type-Classes.md)
* [mongodb/mongo-spark](https://github.com/mongodb/mongo-spark/blob/master/src/main/scala/com/mongodb/spark/NotNothing.scala)
* [Implicit Parameters in Scala](http://baddotrobot.com/blog/2015/07/03/scala-implicit-parameters/)
* [Implicit Parameters](http://docs.scala-lang.org/tutorials/tour/implicit-parameters.html)


# State Action and FSM

`type State[S, +A] = S => (A, S)`

`State` 代表*携带者某种状态的计算*，或 state action，state transition，或 statement。`State` 类型表示将 `S` 从一个状态转换到另一个状态，并产生副作用 `A`。这有点像有限状态机。

考虑一个 Mealy 型的 FSM，根据定义（主要是转移函数和输出函数），把这个 FSM 表示为，

```scala
case class Mealy[S, I, A](t: (S, I) => S, g: (S, I) => A)
```

其中，

* `S`：状态的有限集合
* `I`：输入字母表的有限集合
* `A`：输出字母表的有限集合
* `t`：转移函数
* `g`：输出函数

转移函数和输出函数都接受 `S` 和 `I` 的值作为参数，可以把这两个函数合并，得到，

```scala
(S, I) => (A, S)

// currying
I => S => (A, S)
```

根据 State 的定义得到，`I => State[S, A]`。`State[S, A]` 代表类型为 `S => (A, S)` 的函数，这个函数参数可以看做这个 Mealy 型的 FSM 的初始状态。

同样的一个 Moore 型的 FSM，根据定义可以表示为，

```scala
case class Moore[S, I, A](t: (S, I) => S, g: S => A)
```

转移函数和输出函数都接受 `S` 的值作为参数，合并得到，`S => (I => S, A)`，但此时就不能够用 `State` 数据类型来表示了。

# implicit

* [Implicits mechanism in Scala](http://akmetiuk.com/posts/2017-05-12-implicits.html)
* [Implicit Parameters](http://docs.scala-lang.org/tutorials/tour/implicit-parameters.html)
* [Implicit Classes](http://docs.scala-lang.org/overviews/core/implicit-classes.html)
* [Understanding implicit in Scala](http://stackoverflow.com/questions/10375633/understanding-implicit-in-scala)
* [Implicit Design Patterns in Scala](http://www.lihaoyi.com/post/ImplicitDesignPatternsinScala.html)
