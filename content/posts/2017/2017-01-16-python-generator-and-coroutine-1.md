---
title: Python Generator and Coroutine 1
date: 2017-01-16 15:29:30
categories:
    - concurrency
tags:
    - concurrency
    - python
    - coroutine
---

# Generator（生成器）

在Python中，说到generator，就不得不提iterator和iterable，下面这张图来自[Iterables vs. Iterators vs. Generators](http://nvie.com/posts/iterators-vs-generators/)，这里做个简单的说明（文章很好的解释了这三者的关系与区别）。

![](/images/2017/14839555031473.jpg)

* Container：是一个把元素组织在一起的数据结构，可以判断元素是否包含在容器当中。（大多数）容器提供了一种能够得到他们包含的每个元素的方法，这使得这些容器是iterable（可迭代对象）。
* Iterable：iterable是任何一个可以返回iterator的对象（不限于容器）。
* Iterator：带状态的对象，当对这个对象调用`next()`时，可以产生下一个值。任何有`__next__()`方法的对象都是一个iterator。Iterator就像一个lazy factory，当需要时，才产生一个值返回。

下面是一个iterator的例子，与此同时也是一个iterable，

```python
class fib:
    def __init__(self):
        self.prev = 0
        self.curr = 1

    def __iter__(self):
        return self

    def __next__(self):
        value = self.curr
        self.curr += self.prev
        self.prev = value
        return value
```

下面着重要说的是[generator](http://stackoverflow.com/questions/231767/what-is-the-function-of-the-yield-keyword)，genarator是一种特殊的iterator，因此它是一个惰性求值的factory。继续上面的例子，generator能够避免编写`__iter__()`和`__next__()`，而以一种简洁优雅的方式写出上面的`fib` iterator。

```python
def fib():
    prev, curr = 0, 1
    while True:
        yield curr
        prev, curr = curr, prev + curr

f = fib()
list(islice(f, 0, 10))
```

当执行`fib()`时，实例化并返回了一个generator，除此之外什么代码都没有被执行，包括`prev, curr = 0, 1`。`islice`是一个iterator，因此`fib`的代码仍然未被执行。

而`list`会使用其参数，由其来构造一个list，它会对`islice`对象调用`next()`，进而会对`f`对象调用`next()`，此时才开始执行`fib()`里的代码，直至`yield curr`，返回`curr`中的值，并暂停执行`fib()`，`fib()`的状态被冻结了。这个值被返回给`islice`，最终被添加到list里面。然后重复上述过程，直到产生第10个元素。

generator的类型有函数generator和表达式generator，表达式generator语法类似列表生成式，

```python
numbers = [1, 2, 3, 4, 5, 6]
square_list = [x * x for x in numbers]
square_set = {x * x for x in numbers}

lazy_square_list = (x * x for x in numbers)
```

到目前为止，似乎generator相比起iterator，除了更简洁以外，没有什么特别的东西。[pep-0342](https://www.python.org/dev/peps/pep-0342/)（*A new method for generator-iterators is proposed, called send(). It
takes exactly one argument, which is the value that should be "sent in" to the generator*）.规定了一个genarator可以产生一个值，或者在产生一个值的同时还接收一个值。

```python
def fib():
    prev, curr = 0, 1
    while True:
        old_curr = curr
        curr = yield old_curr
        if not curr is None and not curr == old_curr:
            prev, curr = curr, curr + 1
        else:
            prev, curr = old_curr, prev + old_curr

f = fib()
list(islice(f, 0, 10)) # [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
f.send(100)
list(islice(f, 0, 10)) # [201, 302, 503, 805, 1308, 2113, 3421, 5534, 8955, 14489]
```

第二次的`list(islice(f, 0, 10))`结果不是`[89, 144, 233, 377, 610, 987, 1597, 2584, 4181, 6765]`，因为`send()`改变了curr的值。

在调用`send()`前，由于还未执行到`yield`处，因此必须先调用一次`next()`或`send(None)`。

# Coroutines（协程）

与Coroutines对应的概念是Subroutine（子程序）。一个普通的函数调用是这样的，从函数的第一行执行到`return`语句或exception，或者执行到函数的结尾，这样也叫做一个subroutine。但有时候又希望函数能够生成一系列的值，而不仅仅是返回一个值，此时函数不应该return（return control of execution），而是yield（transfer control temporarily and voluntarily），因为函数需要稍后继续执行。generator能够冻结函数的状态，继续执行的时候恢复。

下面是用generator来实现的一个生产者-消费者模型，

```python
import random

def consume():
    consumed_count = 0
    while True:
        data = yield
        consumed_count += 1
        print('Consuming {}, Total Consumed {}'.format(data, consumed_count))

def produce(consumer):
    while True:
        data = random.random()
        print('Produced {}'.format(data))
        consumer.send(data)
        yield

consumer = consume()
consumer.send(None) # or consumer.next()
producer = produce(consumer)
for _ in range(10):
   print('Producing...')
   next(producer)

# Producing...
# Produced 0.4138693968479813
# Consuming 0.4138693968479813, Total Consumed 1
# Producing...
# Produced 0.5462849666609885
# Consuming 0.5462849666609885, Total Consumed 2
# Producing...
# Produced 0.06190270111408913
# Consuming 0.06190270111408913, Total Consumed 3
```

# References

1. [谈谈Python的生成器](http://www.bjhee.com/python-yield.html) 里有一个关于`send`不错的例子
2. [Improve Your Python: 'yield' and Generators Explained](https://jeffknupp.com/blog/2013/04/07/improve-your-python-yield-and-generators-explained/)
