title: "C++ Concurrency in Action (2) - Managing threads"
date: 2015-05-18 19:35:23
description: Managing threads的笔记。
categories:
    - concurrency
tags:
    - threads
    - reading
    - cpp
    - cpp11
    - 'c++ concurrency in action'
---

# Managing threads

`std::thread`：线程管理的相关类和函数。

对于那些不是那么简单的任务，库提供了可以让你从基本的代码来构建你需要的东西的灵活性。

## Basic thread management

每个C++程序都至少有一个线程，这个线程是由C++运行时启动的：运行`main()`的那个线程。

### Launching a thread

启动一个线程需要构造`std::thread`对象：

```cpp
void do_some_work();
std::thread my_thread(do_some_work); //此时新线程已经启动
```

在C++标准库中，`std::thread`可以适用于任何callable类型。如果是一个带有函数调用操作符的类的实例，那么对象会被copy到新线程的存储空间。

```cpp
class background_task
{
public:
    void operator()() const
    {
        do_something();
        do_something_else();
    }
};

background_task f;
std::thread my_thread(f);
```

但要注意的是，如果传入的是一个临时对象，而不是已经命名的对象，那么这样的语法就和函数调用没有区别，编译器也不会把它看作是callable对象。
例如：

```cpp
std::thread my_thread(background_task());
```

这里声明了一个my_thread函数，接受一个参数，参数是一个函数指针（这个函数不接受任何参数，返回的是background_task对象），返回一个`std::thread`对象。

避免这样问题的方法：
1. 命名函数对象
2. 使用额外的括号
3. 使用新的统一的初始化语法

```cpp
std::thread my_thread((background_task()));    //prevent interpretation as a function
std::thread my_thread{background_task()};
```

#### lambda表达式

```cpp
std::thread my_thread([](
    do_something();
    do_something_else();
});
```

在线程启动以后，需要决定是等待线程结束，还是任其运行。这个决定只需要在线程destroy之前完成即可，因为有可能在你join或detach前，线程就运行完成了。

如果不想等待线程结束，那么必须保证线程所访问的数据，直到该线程结束时都是合法的。下面就是一个反例：

```cpp
struct func
{
    int& i;
    func(int& i_):i(i_){}
    void operator()()
    {
        for(unsigned j=0;j<1000000;++j)
        {
            do_something(i);
        }
    }
};

void oops()
{
    int some_local_state=0;
    func my_func(some_local_state);
    std::thread my_thread(my_func);
    my_thread.detach();
}
```

上面的代码中，因为调用了`detach()`，所以`oops()`结束时，新线程仍然有可能还在运行。如果仍然在运行，那么`do_something(i)`将会访问一个已经destroyed的变量。

一种常用的方式是，使得thread function self-contained，并且是copy数据到线程，（这里指的应该是function object），而不是共享数据（指针或引用）。除非可以保证线程在函数结束前运行完，否则不要创建一个可以访问所在函数局部变量的线程。当然，也可以join。

### Waiting for a thread to complete

对与线程相关联的`std::thread`对象调用`join()`。

在上例中，可以换成`join()`。但是换了以后就失去了多线程的意义，因为原始线程除了wait，什么都没法做。

`join()`是一种简单粗暴的方法。如果需要细粒度的控制wait，那么就需要其他的机制。

调用`join()`还会清除与线程相关的任何storage，因此`std::thread`对象不再和任何已结束的线程关联，换句话说就是，对于给定线程，`join()`只能调用一次。

```cpp
func my_func;
std::thread t(my_func);
t.join();
```

这里加断点可以验证，t在join前，thread id的值不为0，join后变为0。

### Waiting in exceptinal circumstances

`detach()`可以在线程开始后马上调用，但是`join()`意味着wait。如果想要在wait前做些其他事情，那么就必须考虑`join()`放置的位置。

```cpp
struct func;
void f()
{
    int some_local_state=0;
    func my_func(some_local_state);
    std::thread t(my_func);
    try
    {
        do_something_in_current_thread();
    }
    catch(...)
    {
        t.join();
        throw;
    }
    t.join();
}
```

上述代码保证了在异常或者无异常的情况下，都能够`join()`。无论是什么原因导致要`join()`，都必须保证在所有exit可能的情况里，都有`join()`，而上面的代码太复杂，容易出错。

#### RAII

```cpp
class thread_guard
{
    std::thread& t;
public:
    explicit thread_guard(std::thread& t_):
        t(t_)
    {}
    ~thread_guard()
    {
        if(t.joinable())
        {
            t.join();
        }
    }
    thread_guard(thread_guard const&)=delete;
    thread_guard& operator=(thread_guard const&)=delete;
};

struct func;
void f()
{
    int some_local_state=0;
    func my_func(some_local_state);
    std::thread t(my_func);
    thread_guard g(t);
    do_something_in_current_thread();
}
```

但f exit时，局部对象的销毁顺序是与构造顺序相反的。因此无论是什么情况导致f exit，只要t是joinable的，就可以保证join。

之所以要disable copy和assign，是因为如果enalbe，那么对象的生命周期可能会超过thread应该join的作用域。

### Runing threads in the background

在一个`std::thread`对象上调用`detach()`。

一旦调用`detach()`，就再也无法wait for that thread（不能获得reference到that thread的`std::thread`对象，也不能`join()`）。

被`detach()`的线程（也被叫做demon thread）会在后台运行，拥有权和控制权会交给C++ Runtime library，它能保证当线程结束时，相关的资源会被回收。这样的线程可能会是long-running的线程，执行监视、清理和优化的工作。

为了从一个`std::thread`对象上`detach()`线程，必须要有线程来detach。调用`detach()`的要求和`join()`一样，joinable的`std::thread`对象才可以detach()。

## Passing arguments to a thread function

可以用前面的方法，用一个带有data成员的函数对象，但更简便的是：

```cpp
void f(int i,std::string const& s);
std::thread t(f,3,”hello”);
```

在上例中，要注意的是，std::string是以char const*的形式传入的，只有在新线程的context中才会被转为`std::string`。

在默认情况下，参数是被copy的。我猜这样设计的原因也是出于之前提到过的原因，如果线程point to或reference to的local variable所在的scope结束，local variable就会被销毁，那么线程将会访问一个已经destroyed的变量。除非使用额外的`join()`，但这无疑增加了用户编码的复杂度。

### Just want reference

如果我就是要修改原始数据，怎么办？对于pointer，这个倒是好说，直接传pointer即可，地址会被copy，但是reference就不一样了。

```cpp
void update_data_for_widget(widget_id w,widget_data& data);

void oops_again(widget_id w)
{
    widget_data data;
    std::thread t(update_data_for_widget,w,data);
    display_status();
    t.join();
    process_widget_data(data);
}
```

上例中，虽然`update_data_for_widget()`期望的是第二个参数传入引用，但是`std::thread`并不知道。`update_data_for_widget()`被调用时，data实际上是reference to线程内部的copy过来的data，而不是原始的data。线程结束时，这些对data的操作都会随着线程内部copy的销毁而丢失，`process_widget_data()`接受的还是没有修改的data。

但是我在clang++-3.6，libstdc++的环境下编译的时候，以上代码是无法通过编译的，错误如下：

```cpp
error: no type named 'type' in 'std::result_of<void (*(int, double))(int, double &)>'
      typedef typename result_of<_Callable(_Args...)>::type result_type;
```

加入std::ref后编译通过，

```cpp
std::thread t(update_data_for_widget,w,std::ref(data));
```

### `std::thread` and `std::bind`

`std::thread`的构造函数和`std::bind`的操作有相同的机制，可以这样构造`std::thread`对象。

```cpp
class X
{
public:
    void do_lengthy_work();
};
X my_x;
std::thread t(&X::do_lengthy_work,&my_x);
```

如果成员函数有参数，那么可以作为构造函数的第三个参数，以此类推。

### objects cannot be copied

有的对象不能够被copy，比如`std::unique_ptr`对象。这时需要用`std::move()`来transfer ownership到另一个`std::unique_ptr`对象。

```cpp
void process_big_object(std::unique_ptr<big_object>);
std::unique_ptr<big_object> p(new big_object);
p->prepare_data(42);
std::thread t(process_big_object,std::move(p));
```

## Transferring ownership of a thread

虽然`std::thread`不像`std::unique_ptr`动态的拥有一个对象，但是`std::thread`的确是拥有资源：每个`std::thread`实例负责管理一个线程的执行。由于`std::thread`对象不是copyable，而是moveable，因此对象的ownership可以在对象间transfer。

```cpp
void some_function();
void some_other_function();
std::thread t1(some_function);
std::thread t2=std::move(t1);    //t1不再与运行some_function()的线程关联
t1=std::thread(some_other_function);    //如果是临时对象，move自动并且隐式的发生
std::thread t3；    //默认构造，没有和任何执行线程关联
t3=std::move(t2);
t1=std::move(t3);
```

在上例中的最后一个move，t1原本是和运行`some_other_function()`的线程关联的，但是运行着`some_function()`的线程的ownership被transfer给了t1，这将导致程序终止。
因为在线程运行结束并销毁前，要么`join()`，要么`detach()`，但是绝对不能够简单的通过向管理它的`std::thread`对象赋值(move)而丢掉它。“野线程”不允许存在。

```cpp
std::thread f()
{
    void some_function();
    return std::thread(some_function);
}

std::thread g()
{
    void some_other_function(int);
    std::thread t(some_other_function,42);
    return t;
}
```

上例中，实际上是在transfer ownership。

在这里，`std::move`的另一个作用可以简化`thread_gurad`。在原来的`thread_gurad`中，

```cpp
thread_gurad(std::thread(do_work, 2));
```

这样是不允许的，因为`thread_gurad`构造函数接受的参数是引用，因此传入的必须是左值，而unnamed `std::thread` object并不是左值对象。

使用`std::move`后，

```cpp
class scoped_thread {
  std::thread t;
public:
  explicit scoped_thread(std::thread t_) :
      t(std::move(t_)) {
    if (!t.joinable()) {
      throw std::logic_error("No thread");
    }
  }

  ~scoped_thread() {
    t.join();
  }

  scoped_thread(scoped_thread const &p) = delete;

  scoped_thread &operator=(scoped_thread const &) = delete;
};

scoped_thread(std::thread(do_work, 2));
```

上述操作就可以了。这样做还避免了`thread_gurad`对象的生命周期可能超过它引用的线程所在的scope，并且transfer以后，没有其他关联的`std::thread`对象可以join或detach。

## Choosing the number of threads at runtime

`std::thread::hardware_concurrency()`，这个函数返回可以真正并行执行的线程数目。但这只是个hint，换句话说，就算可以并发多个线程，如果没有可用的信息，它可能会返回0。

C++ Concurrency in Action书中，在并行累加例子的后面有并行算法要求的共性的总结：

* at least forward iterators
* single-pass input iterators
* T must be default constructiable

到目前位置，由于不能够直接从线程中返回值，因此必须传入reference。

## Identifying threads

线程识别符是`std::thread::id`类型的，获得方式有：

* 通过成员函数`get_id()`，从关联该线程的`std::thread`对象获得（如果对象没有关联任何线程，则会返回默认构造函数生成的`std::thread::id`对象，表示not any thread）
* 对于当前线程，使用`std::thread::get_id()`


`std::thread::id`对象可以被copy，并且该类型提供了完整的比较操作（全序的）。如果一致，那么他们代表同一线程，或者*not any thread*。该类型对象还可以作为key用于关联容器、排序，同时标准库还提供了`std::hash<std::thread::id>`，因此还能用于无序关联容器。

`std::thread::id`对象常用于检查线程是否需要做某些操作。
