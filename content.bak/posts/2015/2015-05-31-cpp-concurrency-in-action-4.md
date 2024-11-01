title: "C++ Concurrency in Action (4) - Synchronizing concurrent operations"
date: 2015-05-31 13:26:54
description: Synchronizing concurrent operations的笔记。
categories:
    - concurrency
tags:
    - synchronize
    - reading
    - cpp
    - cpp11
    - 'c++ concurrency in action'
---

# C++ Concurrency in Action 4

## Waiting for an event or other condition

1. 一直检查某个flag（被mutex保护）

	消耗资源，被锁住的flag实际上并不能被其他线程访问。消耗资源导致了被等待的线程运行时得到的资源更少，使得等待时间更长。

2. condition variable

	wait and notify

3. `std::this_thread::sleep_for()`

	```cpp
	bool flag;
	std::mutex m;
	void wait_for_flag()
	{
	    std::unique_lock<std::mutex> lk(m);
	    while(!flag)
	    {
	        lk.unlock();
			std::this_thread::sleep_for(std::chrono::milliseconds(100));
	        lk.lock();
	    }
	}
	```
	sleep前unlock，以便其他线程有机会获得lock。但是难以掌握sleep的时长。

### Waiting for a condition with condition variables

* `std::condition_variable`：和mutex一起用，为了提供合适的同步。
* `std::condition_variable_any`

mutex_like即可，但是通用是要付出占用空间、性能或所需资源上的代价的。除非需要额外的灵活度，否则i用前者。

```cpp
std::mutex mut;
std::queue<data_chunk> data_queue;
std::condition_variable data_cond;
void data_preparation_thread() {
    while(more_data_to_prepare()) {
        data_chunk const data=prepare_data();
        std::lock_guard<std::mutex> lk(mut);
        data_queue.push(data);
        data_cond.notify_one(); //notify waiting thread
    }
}

void data_processing_thread() {
    while(true) {
	    std::unique_lock<std::mutex> lk(mut);
	    data_cond.wait(
            lk, []{ return !data_queue.empty(); });
        data_chunk data=data_queue.front();
        data_queue.pop();
        lk.unlock();  //不要在处理的时候（可能耗时）持有
        process(data);
        if(is_last_chunk(data))
            break;
    }
}
```

1. `wait()`，`mut`，predicate有什么关系？

	thread wakes(notified by `notify_one()`) -> lock mutex -> check predicate；

	> predicate false -> unlock mutex -> thread blocked or waiting；
	> predicate true -> still leave mutex locked -> return from `wait()`；

2. 为什么在`data_processing_thread()`要使用`std::unique_lock`？

	反过来考虑为什么不是`std::lock_guard`和`std::mutex`。

	`std::lock_guard`直到销毁才会unlock，但是线程wait的时候，就隐式的进行了unlock。

	`std::mutex`，[这里](http://stackoverflow.com/questions/13099660/c11-why-does-stdcondition-variable-use-stdunique-lock)从API设计的角度上解释了为什么不是。

在调用`wait()`期间，predicate会在mutex locked的条件下，被检查任意多次。当且仅当predicate true， `wait()`立即返回。

**spurious wake**

当等待线程重新获取锁并检查条件时，如果它不直接响应另一个线程的notification（例如：你的predicate和共享的变量无关，另一个线程notify的时候，predicate就不会直接respond），这就是spurious wake。是有side effect的。

**`notify_one()`和`notify_all()`**

* 当有多个线程wait同一个event的时候，`notify_one()`并不能保证哪个线程会被通知。
* 当有多个线程wait同一个event，并且所有线程都需要respond的时候，`notify_all()`会导致这些线程都去check predicate。

## Waiting for one-off events with futures

`std::future`：provides a mechanism to access the result of asynchronous operations。

由`std::unique_ptr`和`std::shared_ptr`建立：

1. unique futures：`std::future<>`

	* moveable only
	* 其实例是唯一关联到与它关联事件的实例。
	* ownership可以在实例间transfer，但是有一个实例可以引用到特定异步操作的结果。

2. shared futures：`std::shared_future<>`

	* copyable
	* 其多个实例可以指向同一个事件。在这个情况下，所有的实例都会同时ready，都可以访问与事件关联的数据。
	* 可以有多个实例引用到关联状态。

如果无关联的数据，用`std::future<void>`或`std::shared_future<void>`。

### Returning values from background tasks

可以用`std::async()`开始一个**异步任务**。`std::async()`返回一个`std::future`对象，这个对象最终将持有函数的返回值，用`std::future`的`get()`获得（线程会block到future ready）。

对`std::async()`提供参数类似于`std::thread`和`std::call_once`。

```cpp
struct X {
    void foo(int,std::string const&);
    std::string bar(std::string const&);
};
X x;
auto f1=std::async(&X::foo,&x,42,"hello");
auto f2=std::async(&X::bar,x,"goodbye");

struct Y {
    double operator()(double);
};
Y y;
auto f3=std::async(Y(),3.141);
auto f4=std::async(std::ref(y),2.718);

X baz(X&);
std::async(baz,std::ref(x));

auto f5=std::async(move_only()); //对于右值，std::move会被隐式的调用
```

`std::async()`的运行由实现决定（自己试了发现libstdc++6用的是`std::launch::deferred`），但也可以由参数（`std::launch`类型）指定。

* `std::launch::deferred`：函数调用推迟到在future上调用`wait()`或`get()`。
* `std::launch::async`：函数在它自己的线程上运行。
* `std::launch::deferred | std::launch::async`：默认，视实现而定。

如果函数调用推迟了，那么它可能再也不会实际执行。

通过修改参数并运行下面这段代码，来体会它们的区别，

```cpp
timer t; //可以自己简单的实现

std::future<int> the_answer = std::async(std::launch::deferred, []() {
  for (int i = 0; i < 999999999; ++i) {
    int a = 0;
  }
  return 1 + 1;
});
std::cout << "futuring..." << std::endl;
for (int i = 0; i < 999999999; ++i) {
  int a = 0;
}
std::cout << the_answer.get() << std::endl;
}
```

可以发现使用`std::launch::deferred`的时间，几乎是`std::launch::async`的两倍，这里就能很形象的感受**defer**。

### Associating a task with a future

`std::packaged_task<>`把一个函数或callable对象绑定到一个future，当`std::packaged_task<>`被调用的时候，它进而调用关联的函数或callable对象使得future ready，返回值作为关联数据储存。

`std::packaged_task`不是copyable，但是moveable。

`std::packaged_task`的模板参数是函数签名。构造实例时，传入的callable对象要能接受指定的参数，并且返回值类型可以转换到所指定的返回类型。也就是说不必100% match，但是至少也要保证可以隐式转换。

> * 函数签名的返回值类型指定了从`get_future()`返回的`std::future`的类型。
> * 函数签名的参数列表指定了，封装的任务的函数调用的签名。

```cpp
auto f = [](int b) {
  for (int i = 0; i < 999999999; ++i) {
    int a = 0;
  }
  return b;
};

std::packaged_task<int(int)> task(f);
std::future<int> res = task.get_future();
//res.wait();  //waiting endlessly
//std::cout << res.get() << std::endl;  //waiting endlessly
task(2);
std::cout << res.get() << std::endl;
```

* 上例中，在`task()`调用前，绑定的函数是没有执行的，因此这里无论是对future进行wait或是get，都是无限的等下去。
* 调用`task()`时，任务其实是在当前的线程中执行的，不会新建一个线程执行。

### Making (std::)promises

`std::promise<T>`提供了一种设置值的方式，这个值可以稍后被关联的`std::future<T>`对象读取。等待线程会在future上block，提供数据的线程可以用promise的`set_value()`来设置值，使得future ready。如果没有设置值就销毁`std::promise`，那么exception将会被存储。

```cpp
auto f = [](int b) {
  for (int i = 0; i < 999999999; ++i) {
    int a = 0;
  }
  return b;
};

std::promise<int> p;
p.set_value(f(2));
auto fu = p.get_future();
std::cout << fu.get() << std::endl;
```

### Saving an exception for the future

1. `std::async`

	就像直接调用函数一样，

	> 函数抛出exception -> exception被存储在future中，替代所存储的值 -> future ready -> `get()`会再次抛出exception
但是`get()`抛出的exception是原始的对象或copy，标准没有规定。

2. `std::packaged_task`

	* 类似`std::async`，

	> 调用task -> 函数抛出exception -> exception被存储在future中，替代所存储的值 -> future ready -> `get()`会再次抛出exception

	* 直接destory `std::packaged_task`

	> 如果future没有ready -> destructor存储`std::future_error` exception在关联的状态中。
	> error code = `std::future_errc::broken_promise`

3. `std::promise`

	* 类似前两者，但是需要显式的函数调用。如果要存储的不是值，是exception，就要调用`set_exception()`。

	* 通常在try/catch中使用，
	```cpp
	extern std::promise<double> some_promise;
	try {
	    some_promise.set_value(calculate_value());
	}
	catch(...) {
	    some_promise.set_exception(std::current_exception());
	}

	some_promise.set_exception(std::copy_exception(std::logic_error("foo")));
	```

	> 上例使用了`std::current_exception()`来获取已引发的异常。还可以	用`std::copy_exception()`创建新的exception，在exception已知的情况下，这样更简洁。

	* 直接destroy `std::promise`

	> 如果future没有ready -> destructor存储`std::future_error` exception在关联的状态中。
	> error code = `std::future_errc::broken_promise`

**关于destory `std::packaged_task`和`std::promise`**

* `std::packaged_task`是`std::promise`更高层次的抽象，所以直接destroy以后，它们的行为是很相似的。
* 创建了future，你就promise to provide一个值或exception，如果你摧毁了他们的来源，你就break了promise。如果destructor不存储`std::future_error` exception，等待future的线程就会一直等下去。

### Waiting from multiple threads

在多个线程中访问`std::future`会有data race和undefined behavior。

原因：

> by design. It models unique ownership of the asynchronous result。因此并发的访问的没意义的，`get()`只能被调用一次。

1. `std::shared_future`

	就算有`std::shared_future`，特定对象的成员函数还是不同步的，要使用lock来避免data race，或者在每个线程创建并访问自己的copy。

2. 构造`std::shared_future`

	引用异步状态的`std::shared_future`实例是由引用了这些状态的`std::future`实例构造的。

	```cpp
	std::promise<int> p;
	std::future<int> f(p.get_future());
	// f refers to asynchronous state of p
	std::shared_future<int> sf(std::move(f));
	// 现在f是invalid


	// 或者
	std::shared_future<int> sf(p.get_future());
	auto sf = p.get_future().share();  // transfer ownership directly
	```

## Waiting with a time limit

two sorts of timeouts:

* duration-based timeout
* absolute timeout

### Clocks

1. `system_clock`

	Wall clock time from the system-wide real-time clock.

	`std::chrono::system_clock::now()`返回系统当前时间，类型是`std::chrono::system_clock::time_point`。

	提供了与`time_t`类型相互转化的函数。

2. `steady_clock`

	* Values of `time_point` never decrease as physical time advances;
	* Values of `time_point` advance at a steady rate relative to real time.

	That is, the clock may not be adjusted.

	`is_steady`可以检测是否是。

3. `high_resolution_clock`

	Clocks with the shortest tick period. `high_resolution_clock` may be a synonym for `system_clock` or `steady_clock`.

**the tick period of clock**

可由clock的`period`成员得到。例如：每秒25次 tick，则是`std::ratio<1, 25>`。

并不能够保证，$在一次运行中观察到的tick period=那个clock指定的period$。

### Durations

`std::chrono::duration<>`

```cpp
template<typename _Rep, typename _Period>
  struct duration
  {
typedef _Rep			rep; // the type of representation
typedef _Period 		period;  // 指定duration的每个unit代表多长时间
...
  }
```

标准库预定义了很多种durations：

```cpp
typedef duration<int64_t, nano> 	    nanoseconds;
typedef duration<int64_t, micro> 	    microseconds;
typedef duration<int64_t, milli> 	    milliseconds;
typedef duration<int64_t> 		    seconds;
typedef duration<int64_t, ratio< 60>>   minutes;
typedef duration<int64_t, ratio<3600>>  hours;
```

它们都使用足够大的整数类型。当然也可以使用预定义的ratio或自己定义的来定义新的duration。

当不会发生截断的时候，durations之间的转换是隐式的。显示的转换可用`duration_cast`，

```cpp
std::chrono::milliseconds ms(54802);
std::chrono::seconds s=
    std::chrono::duration_cast<std::chrono::seconds>(ms);
//截断，而非四舍五入，s=54
```

durations可以与一个常数（`_Rep`类型的）进行加减乘除。

在一个duration中，要知道units的数目，可以调用`count()`。

**基于duration的wait**

之前提到的所有的blocking call都是block一个不确定长度的时间。

当你将duration用于wait，wait会返回一个状态，来标识是超时，还是其他情况。

* `std::future_status::timeout`：the wait times out
* `std::future_status::ready`：the future is ready
* `std::future_status::deferred`：the future is deferred

基于duration的wait的时间是通过一个内部的steady clock来衡量的，但是因为调度，或者精度的原因，实际等待的时间可能会略长。

### Time points

`std::chrono::time_points<>`：存储了从clock的epoch开始的时长（某个duration的倍数）。

```cpp
template<typename _Clock, typename _Dur>
  struct time_point
  {
typedef _Clock		clock; // clock的类型
typedef _Dur		duration;  // 度量从epoch开始的时间
  }
```

对于一个time point，`time_since_epoch()`返回了从clock的epoch，到那个time point的时长。

* 可以将duration与time point相加减，得到新的time point。
* 可以将两个share same clock的time point相减，得到duration。

**基于time point的wait**

在condition variable中，如果不想向`wait()`传入一个predicate，那么最好是使用`wait_until()`，这样循环的总长度（看4.1.1）有限的。

### Functions that accept timeouts

|Class/Namespace|Functions|Return values|
|-|-|-|
|`std::this_thread` namespace|`sleep_for(duration)`, `sleep_until(time_point)`|N/A|
|`std::condition_variable` or `std::condition_variable_any`|`wait_for(lock, duration)`, `wait_until(lock, time_point)`|`std::cv_status::timeout` or `std::cv_status::no_timeout`|
||`wait_for(lock, duration, predicate)`, `wait_until(lock, time_point, predicate)`|bool—the return value of the predicate when awakened|
|`std::timed_mutex` or `std::recursive_ timed_mutex`|`try_lock_for(duration)`, `try_lock_until(time_point)`|bool—true if the lock was acquired, false otherwise|
|`std::unique_ lock<TimedLockable>`|`unique_lock(lockable, duration)`, `unique_lock(lockable, time_point)`|N/A—`owns_lock()` on the newly constructed object; returns true if the lock was acquired, false otherwise|
||`try_lock_for(duration)`, `try_lock_until(time_point)`|bool—true if the lock was acquired, false otherwise|
|`std::future<ValueType>` or `std::shared_ future<ValueType>`|`wait_for(duration)`, `wait_until (time_point)`|`std::future_status::timeout` if the wait timed out, `std::future_ status::ready` if the future is ready, or `std::future_status::deferred` if the future holds a deferred function that hasn't yet started|

# References
1. [Difference between std::system_clock and std::steady_clock?](http://stackoverflow.com/questions/13263277/difference-between-stdsystem-clock-and-stdsteady-clock)
2. C++ Concurrency in Action
