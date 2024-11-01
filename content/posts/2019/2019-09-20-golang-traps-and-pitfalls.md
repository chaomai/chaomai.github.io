---
title: go踩坑合集
date: 2019-09-20 17:07:39
categories:
    - golang
tags:
    - golang
---

# RWMutex RLock重入导致死锁
RWMutex，即读写锁，可以被多个的reader或一个writer获取使用。

## 死锁例子
在使用RWMutex的时候，同一个reader是不应该连续调用`Rlock`多次的，这样做不但没有意义，还有可能导致死锁，具体代码如下：

```go
func main() {
	var l = sync.RWMutex{}
	var wg sync.WaitGroup
	wg.Add(2)

	c := make(chan int)
	go func() {
		l.RLock()
		defer l.RUnlock()
		c <- 1
		runtime.Gosched()

		l.RLock()
		defer l.RUnlock()
		wg.Done()
	}()

	go func() {
		<-c
		l.Lock()
		defer l.Unlock()
		wg.Done()
	}()

	wg.Wait()
}
```

## sync.RWMutex分析
下面[RWMutex的实现](https://github.com/golang/go/blob/master/src/sync/rwmutex.go)，我们来看这段代码的具体执行。为了方便理解，把`if race.Enabled {...}`的相关代码都去除了。

1. **goroutine 1**：`l.RLock()`

    ```go
    func (rw *RWMutex) RLock() {
    	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
    		// A writer is pending, wait for it.
    		runtime_SemacquireMutex(&rw.readerSem, false, 0)
    	}
    }
    ```

    执行`l.RLock()`后，goroutine 1获得写锁。

    * 状态：获得读锁
    * readerCount = 1，readerWait = 0

2. **goroutine 2**：`l.Lock()`

    ```go
    func (rw *RWMutex) Lock() {
    	// First, resolve competition with other writers.
    	rw.w.Lock()
    	// Announce to readers there is a pending writer.
    	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    	// Wait for active readers.
    	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
    		runtime_SemacquireMutex(&rw.writerSem, false, 0)
    	}
    }
    ```

    由于goroutine 1已获得写锁，此时goroutine 2等待。

    * 状态：等待reader释放读锁
    * readerCount = 1 - rwmutexMaxReaders，readerWait = 1

3. **goroutine 1**：`l.RLock()`

    ```go
    func (rw *RWMutex) RLock() {
    	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
    		// A writer is pending, wait for it.
    		runtime_SemacquireMutex(&rw.readerSem, false, 0)
    	}
    }
    ```

    goroutine 1发现readerCount为负，认为有writer获得了写锁，接着也进入了等待状态。

    * 状态：等待
    * readerCount = 2 - rwmutexMaxReaders，readerWait = 1

最后goroutine 1和goroutine 2都进入了等待状态。

## 总结
1. readerCount的作用？
    持有读锁的reader数。置为负时，代表了writer正在或者已经获得了读锁，此时其他reader不能再获得写锁。
2. readerWait的作用，以及在`Lock()`中，为何需要同时判断`r != 0`和`atomic.AddInt32(&rw.readerWait, r) != 0`？
    置readerCount为负的时候，获得了写锁，但尚未RULock的reader数。writer需要等待这些reader执行结束。

    * 若`r == 0`，则无正在持有读锁的reader，可以直接完成读锁的加锁。
    * 若`r != 0`，writer需要等待获得了写锁，但尚未RULock的reader执行结束。如何判断是否需要等待呢，即readerWait不为0的时候。同时reader决定是否能唤醒writer，也需要等到readerWait为0的时候。

    ```go
    func (rw *RWMutex) Lock() {
    	rw.w.Lock()
    	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    	// 此刻起，其他reader不再能够获得读锁。
    	// 此时，尚未释放写锁的reader数为readerWait个，等待他们结束才能完成读锁的加锁。
    	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
    		runtime_SemacquireMutex(&rw.writerSem, false)
    	}
    }
    ```

# sync.WaitGroup使用的问题
## 例子
在实现一个需求的时候，需要等待一定数目的go协程执行完毕，但这个数目事先并不好确定。想到了可以用sync.WaitGroup来完成，在使用时候发现，`Wait()`没有生效，并未等待协程结束，代码大致如下，

```go
func main() {
	wg := sync.WaitGroup{}
	wg.Add(1)
	ch := make(chan struct{})

	f := func(){
		wg.Add(1)
		defer wg.Done()
		select {
			case <- ch:
				fmt.Println("hi")
		}
	}

	go func() {
		defer wg.Done()
		go f()
		go f()
		go f()
	}()
	close(ch)
	wg.Wait()
}
```

执行后程序不会有任何的输出就退出了。

## sync.WaitGroup源码分析
> Typically this means the calls to Add should execute before the statement creating the goroutine or other event to be waited for.

## 总结
```go
func main() {
	wg := sync.WaitGroup{}
	wg.Add(1)
	ch := make(chan struct{})

	f := func(){
		defer wg.Done()
		select {
			case <- ch:
				fmt.Println("hi")
		}
	}

	go func() {
		defer wg.Done()
		wg.Add(1)
		go f()
		wg.Add(1)
		go f()
		wg.Add(1)
		go f()
	}()
	close(ch)
	wg.Wait()
}
```

## References
1. [Waiting on an indeterminate number of goroutines](https://stackoverflow.com/questions/18805416/waiting-on-an-indeterminate-number-of-goroutines)

# goroutine
```go
func process(ctx context.Context, wg *sync.WaitGroup, retCh chan int) {
	defer wg.Done()
	for {
		time.Sleep(time.Second*1)
		// process
		// send process result back
		select {
			case <-ctx.Done():
				retCh <- 1
				retCh <- 2
				return
			default:
				retCh <- 1
				retCh <- 2
		}
	}
}

func loop(ctx context.Context, wg *sync.WaitGroup, retCh chan int) {
	defer wg.Done()
	wg.Add(1)
	go process(ctx, wg, retCh)
	for {
		select {
			case <-ctx.Done():
				return
			case ret := <-retCh:
				fmt.Println(ret)

		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	wg := &sync.WaitGroup{}
	retCh := make(chan int, 1)
	wg.Add(1)
	go loop(ctx, wg, retCh)

	time.Sleep(time.Second*5)
	cancel()
	wg.Wait()
}
```
