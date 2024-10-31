# Go面试题

## Go语言特点

Go语言相比C++/Java等语言是优雅且简洁的，是我最喜爱的编程语言之一，它既保留了C++的高性能，又可以像Java，Python优雅的调用三方库和管理项目，同时还有接口，自动垃圾回收和goroutine等让人拍案叫绝的设计。

有许多基于Go的优秀项目。**Docker，Kubernetes，etcd，deis，flynn，lime，revel等等。Go无疑是云时代的最好语言！**

题外话到此为止，在面试中，我们需要深入了解Go语言特性，并适当辅以源码阅读：Go源码非常人性化，注释非常详细，基本上只要你学过Go就能看懂）来提升能力。常考的点包括：切片，通道，异常处理，Goroutine，GMP模型，字符串高效拼接，指针，反射，接口，sync，go test和相关工具链。

## 01 切片与数组的区别

在Go语言中，数组（Array）和切片（Slice）是两种不同的数据类型，它们有以下区别：

1. 长度固定 vs 动态长度：

2. - 数组是长度固定的，在声明时需要指定长度，并且无法改变长度。
   - 切片是动态长度的，可以根据需要自动调整长度，无需在声明时指定长度。

3. 值类型 vs 引用类型：

4. - 数组是值类型，赋值或传递数组时会进行值的复制。
   - 切片是引用类型，赋值或传递切片时会共享底层数据，修改其中一个切片会影响其他引用该底层数组的切片。

5. 内存分配：

6. - 数组在声明时会直接分配连续的内存空间，长度固定。
   - 切片是基于数组的动态长度的视图，底层使用数组来存储数据，但可以根据需要动态调整切片的长度。

7. 灵活性：

8. - 数组的长度固定，无法动态增加或缩小，需要重新创建一个新的数组。
   - 切片可以通过追加元素或切割操作来动态增加或缩小长度。

9. 使用场景：

10. - 数组适用于固定长度的数据集合，如存储一组固定大小的元素。
    - 切片适用于动态长度的数据集合，如存储可变数量的元素，并且经常需要进行动态调整。

总的来说，数组适用于长度固定的数据集合，而切片适用于动态长度的数据集合。切片提供了更大的灵活性和方便的操作，是在Go语言中更常用的数据结构。

## 02 切片的使用方法

### 初始化

1. make函数初始化

```go
s := make([]int, 0)
```

1. 从数组中截取

```go
arr := [4]int{0, 1, 2, 3}
s := arr[2:3] // s: [2]
```

### 获取长度和容量

```go
len(s) // 获取长度
cap(s) // 获取容量
```

### 添加元素

```go
arr := [4]int{0, 1, 2, 3}
s := arr[2:3] // s: [2]
s = append(s, 3, 4, 5, 6) // s: [2,3,4,5,6]
```

### 删除元素

```go
s = append(s[:2], s[3:]...) // s: [2,3,5,6]
```

### 遍历

```go
for k, v := range s {
  fmt.Printf("%d:%d\n", k, v)
}
```

## 03 切片的扩容机制

### 1. Go1.18版本前

新申请的容量如果大于当前容量的两倍，会将新申请的容量直接作为新的容量，如果新申请的容量小于当前容量的两倍，会有一个阈值，即当前切片容量小于1024时，切片会将当前容量的2倍作为新申请的容量，如果大于等于1024，会将当前的容量的1.25倍作为新申请的容量。

**源码片段**

```go
 newcap := old.cap
 doublecap := newcap + newcap
 if cap > doublecap {
  newcap = cap
 } else {
  if old.cap < 1024 {
   newcap = doublecap
  } else {
   // Check 0 < newcap to detect overflow
   // and prevent an infinite loop.
   for 0 < newcap && newcap < cap {
    newcap += newcap / 4
   }
   // Set newcap to the requested cap when
   // the newcap calculation overflowed.
   if newcap <= 0 {
    newcap = cap
   }
  }
 }
```

### 2. Go 1.18版本后

新申请的容量如果大于当前容量的两倍，会将新申请的容量直接作为新的容量，如果新申请的容量小于当前容量的两倍，会有一个阈值，即当前切片容量小于256时，切片会将当前容量的2倍作为新申请的容量，如果大于等于256，会将当前的容量的1.25倍+192作为新申请的容量，扩容的时候更加平滑，不会出现从2到1.25的突变。

**源码片段**

```go
newcap := old.cap
 doublecap := newcap + newcap
 if cap > doublecap {
  newcap = cap
 } else {
  const threshold = 256
  if old.cap < threshold {
   newcap = doublecap
  } else {
   // Check 0 < newcap to detect overflow
   // and prevent an infinite loop.
   for 0 < newcap && newcap < cap {
    // Transition from growing 2x for small slices
    // to growing 1.25x for large slices. This formula
    // gives a smooth-ish transition between the two.
    newcap += (newcap + 3*threshold) / 4
   }
   // Set newcap to the requested cap when
   // the newcap calculation overflowed.
   if newcap <= 0 {
    newcap = cap
   }
  }
 }
```

## 04 锁

### 互斥锁（Mutex）

Mutex是Golang的互斥锁，作用是在并发程序中对共享资源的保护，避免出现数据竞争问题。

使用方法：Mutex实现了Locker接口

```go
type Locker interface {
    Lock()
    Unlock()
}
```

也就是互斥锁 Mutex 提供两个方法 Lock 和 Unlock

```go
  func(m *Mutex)Lock()
  func(m *Mutex)Unlock()
```

使用示例：

```go
package main

    import (
        "fmt"
        "sync"
    )

    func main() {
        // 互斥锁保护计数器
        var mu sync.Mutex
        // 计数器的值
        var count = 0
        
        var wg sync.WaitGroup
        wg.Add(10)

        // 启动10个gourontine
        for i := 0; i < 10; i++ {
            go func() {
                defer wg.Done()
                // 累加10万次
                for j := 0; j < 100000; j++ {
                    mu.Lock()
                    count++
                    mu.Unlock()
                }
            }()
        }
        wg.Wait()
        fmt.Println(count)
    }
```

#### Mutex的两种模式

Mutex 可能处于两种操作模式下：正常模式和饥饿模式

**正常模式**

在正常模式下，所有的goroutine会按照先进先出的顺序进行等待，被唤醒的goroutine不会直接持有锁，会和新进来的锁进行竞争，新请求进来的锁会更容易抢占到锁，因为正在CPU上运行，因此刚唤醒的goroutine可能会竞争失败，回到队列头部；如果队列的goroutine超过1毫秒的等待时间，则会转换到饥饿模式。

**饥饿模式**

在饥饿模式下，锁会直接交给队列的第一个goroutine，新进来的goroutine不会抢占锁也不会进入自旋状态，直接进入队列尾部；如果当前goroutine已经是队列的最后一个或者当前goroutine等待时间小于1毫秒，则会转换到正常模式

**正常模式下，性能更好，但饥饿模式解决取锁公平问题，性能较差。**

#### 底层结构

```
 type Mutex struct {
      state int32
      sema  uint32
  }


  const (
      mutexLocked = 1 << iota // mutex is locked
      mutexWoken
      mutexWaiterShift = iota
  )
```

state 是一个复合型的字段，一个字段包含多个意义:

mutexWaiters  阻塞等待的waiter数量

mutexStarving 饥饿标记

mutexWoken 唤醒标记

mutexLocked 持有锁的标记

#### 易错场景

1. Lock/Unlock没有成对出现（加锁后必须有解锁操作），如果Lock之后，没有Unlock会出现死锁的情况，或者是因为 Unlock 一个未Lock的 Mutex 而导致 panic
2. 复制已经使用过的Mutex，因为复制了已经使用了的Mutex，导致锁无法使用，程序处于死锁的状态
3. 重入锁，Mutex是不可重入锁，如果一个线程成功获取到这个锁。之后，如果其它线程再请求这个锁，就会处于阻塞等待的状态
4. 死锁，两个或两个以上的goroutine争夺共享资源，互相等待对方的锁释放

## 读写锁（RWMutex）

RWMutex 是一个 reader/writer 互斥锁。RWMutex 在某一时刻只能由任意数量的 reader goroutine 持有，或者是只被单个的 writer goroutine 持有，适用于读多写少的场景。

### 使用方法

- Lock/Unlock：写操作时调用的方法
- RLock/RUnlock：读操作时调用的方法
- RLocker：这个方法的作用是为读操作返回一个 Locker 接口的对象。它的 Lock 方法会调用 RWMutex 的 RLock 方法，它的 Unlock 方法会调用 RWMutex 的 RUnlock 方法。

使用示例：

```
func main() {
    var counter Counter
    for i := 0; i < 10; i++ { // 10个reader
        go func() {
            for {
                counter.Count() // 计数器读操作
                time.Sleep(time.Millisecond)
            }
        }()
    }

    for { // 一个writer
        counter.Incr() // 计数器写操作
        time.Sleep(time.Second)
    }
}
// 一个线程安全的计数器
type Counter struct {
    mu    sync.RWMutex
    count uint64
}

// 使用写锁保护
func (c *Counter) Incr() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}

// 使用读锁保护
func (c *Counter) Count() uint64 {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.count
}
```

### 底层结构

```
type RWMutex struct {
  w           Mutex   // 互斥锁解决多个writer的竞争
  writerSem   uint32  // writer信号量
  readerSem   uint32  // reader信号量
  readerCount int32   // reader的数量（以及是否有 writer 竞争锁）
  readerWait  int32   // writer等待完成的reader的数量
}

const rwmutexMaxReaders = 1 << 30
```

### 实现原理

一个 writer goroutine 获得了内部的互斥锁，就会反转 readerCount 字段，把它从原来的正整数 readerCount(>=0) 修改为负数（readerCount - rwmutexMaxReaders），让这个字段保持两个含义（既保存了 reader 的数量，又表示当前有 writer）。也就是说当readerCount为负数的时候表示当前writer goroutine持有写锁中，reader goroutine会进行阻塞。

当一个 writer 释放锁的时候，它会再次反转 readerCount 字段。可以肯定的是，因为当前锁由 writer 持有，所以，readerCount 字段是反转过的，并且减去了 rwmutexMaxReaders 这个常数，变成了负数。所以，这里的反转方法就是给它增加 rwmutexMaxReaders 这个常数值。

### 易错场景

1. 复制已经使用的读写锁，会把它的状态也给复制过来，原来的锁在释放的时候，并不会修改你复制出来的这个读写锁，这就会导致复制出来的读写锁的状态不对，可能永远无法释放锁
2. 重入导致死锁，因为读写锁内部基于互斥锁实现对 writer 的并发访问，而互斥锁本身是有重入问题的，所以，writer 重入调用 Lock 的时候，就会出现死锁的现象
3. 在 reader 的读操作时调用 writer 的写操作（调用 Lock 方法），那么，这个 reader 和 writer 就会形成互相依赖的死锁状态
4. 当一个 writer 请求锁的时候，如果已经有一些活跃的 reader，它会等待这些活跃的 reader 完成，才有可能获取到锁，但是，如果之后活跃的 reader 再依赖新的 reader 的话，这些新的 reader 就会等待 writer 释放锁之后才能继续执行，这就形成了一个环形依赖：writer 依赖活跃的 reader -> 活跃的 reader 依赖新来的 reader -> 新来的 reader 依赖 writer
5. 释放未加锁的 RWMutex，和互斥锁一样，Lock 和 Unlock 的调用总是成对出现的，RLock 和 RUnlock 的调用也必须成对出现。Lock 和 RLock 多余的调用会导致锁没有被释放，可能会出现死锁，而 Unlock 和 RUnlock 多余的调用会导致 panic

## 05 死锁

### 概念

两个或两个以上的进程（或线程，goroutine）在执行过程中，因争夺共享资源而处于一种互相等待的状态，如果没有外部干涉，它们都将无法推进下去，此时，我们称系统处于死锁状态或系统产生了死锁。

### 产生死锁的四个必要条件

1. 互斥：资源只能被一个goroutine持有，其他gouroutine必须等待，直到资源被释放
2. 持有和等待：goroutine 持有一个资源，并且还在请求其它 goroutine 持有的资源
3. 不可剥夺：资源只能由持有它的 goroutine 来释放
4. 环路等待：多个等待goroutine（g1,g2,g3），g1等待g2的资源，g2等待g3的资源，g3等待g1的资源，形成环路等待的死结

### 如何解决死锁？（只需要打破必要条件其中一个即可避免死锁）

1. 设置超时时间
2. 避免使用多个锁
3. 按照规定顺序申请锁
4. 死锁检测

## 06 sync.Cond

Cond 通常应用于等待某个条件的一组 goroutine，等条件变为 true 的时候，其中一个 goroutine 或者所有的 goroutine 都会被唤醒执行。

### 基本方法

```
func NeWCond(l Locker) *Cond 
func (c *Cond) Broadcast() 
func (c *Cond) Signal() 
func (c *Cond) Wait()
```

- Singal(): 唤醒一个等待此 Cond 的 goroutine
- Broadcast(): 唤醒所有等待此 Cond 的 goroutine
- Wait(): 放入 Cond 的等待队列中并阻塞，直到被 Signal 或者 Broadcast 的方法从等待队列中移除并唤醒，使用该方法是需要搭配满足条件

使用示例：

```
func main() {
    c := sync.NewCond(&sync.Mutex{})
    var ready int

    for i := 0; i < 10; i++ {
        go func(i int) {
            time.Sleep(time.Duration(rand.Int63n(10)) * time.Second)

            // 加锁更改等待条件
            c.L.Lock()
            ready++
            c.L.Unlock()

            log.Printf("运动员#%d 已准备就绪\n", i)
            // 广播唤醒所有的等待者
            c.Broadcast()
        }(i)
    }

    c.L.Lock()
    for ready != 10 {
        c.Wait()
        log.Println("裁判员被唤醒一次")
    }
    c.L.Unlock()

    //所有的运动员是否就绪
    log.Println("所有运动员都准备就绪。比赛开始，3，2，1, ......")
}
```

### 实现原理

```
type Cond struct {
    noCopy noCopy

    // 当观察或者修改等待条件的时候需要加锁
    L Locker

    // 等待队列
    notify  notifyList
    checker copyChecker
}

func NewCond(l Locker) *Cond {
    return &Cond{L: l}
}

func (c *Cond) Wait() {
    c.checker.check()
    // 增加到等待队列中
    t := runtime_notifyListAdd(&c.notify)
    c.L.Unlock()
    // 阻塞休眠直到被唤醒
    runtime_notifyListWait(&c.notify, t)
    c.L.Lock()
}

func (c *Cond) Signal() {
    c.checker.check()
    runtime_notifyListNotifyOne(&c.notify)
}

func (c *Cond) Broadcast() {
    c.checker.check()
    runtime_notifyListNotifyAll(&c.notify）
}
```

在上述的实现源码中，Signal和Broadcast调用了底层的通知方法；重点在Wait方法中，把调用者加入到等待队列时会释放锁，在被唤醒之后还会请求锁。在阻塞休眠期间，调用者是不持有锁的，这样能让其他 goroutine 有机会检查或者更新等待变量，因此在使用Wait方法的时候必须持有锁。

### 易错场景

1. 调用Wait方法没有加锁
2. 没有检查等待条件是否满足

## 07 Channel

channel用于goroutine之间的通信，go语言中，CSP并发模型，不要通过共享内存实现通信，而是通过通信实现共享内存，就是由goroutine和channel实现的。

### 应用场景

- 数据交流
- 信号通知
- 任务编排
- 锁

### 基本用法

初始化

```
ch := make(chan int, 1) //有缓冲区
ch := make(chan int) //无缓冲区
```

发送数据

```
ch <- 2
```

接收数据

```
x := <-ch // 把接收的一条数据赋值给变量x 
foo(<-ch) // 把接收的一个的数据作为参数传给函数 
<-ch // 丢弃接收的一条数据
```

返回容量

```
c := cap(ch)
```

返回channel中缓存的还未被取走的元素数量

```
l := len(ch)
```

关闭channel

```
close(ch)
```

遍历channel

```
// 第一种
var ch = make(chan int, 10) 
for i := 0; i < 10; i++ { 
  select { 
  case ch <- i: 
  case v := <-ch: fmt.Println(v) 
  } 
}

// 第二种
for v := range ch { 
  fmt.Println(v) 
}
```

### 底层结构

qcount 已经接收但还未被取走的元素个数 内置函数len获取到

datasiz 循环队列的大小  暂时认为是cap容量的值

elemtype和elemsize 声明chan时到元素类型和大小 固定

buf 指向缓冲区的指针  无缓冲通道中 buf的值为nil

sendx 处理发送进来数据的指针在buf中的位置 接收到数据 指针会加上elemsize，移向下一个位置

recvx 处理接收请求（发送出去）的指针在buf中的位置

recvq  如果没有数据可读而阻塞， 会加入到recvq队列中

sendq 向一个满了的buf 发送数据而阻塞，会加入到sendq队列中

### 实现原理

*向channel写数据的流程*：

有缓冲区：优先查看recvq是否为空，如果不为空，优先唤醒recvq的中goroutine，并写入数据；如果队列为空，则写入缓冲区，如果缓冲区已满则写入sendq队列；

无缓冲区：直接写入sendq队列

*向channel读数据的流程*：

有缓冲区：优先查看缓冲区，如果缓冲区有数据并且未满，直接从缓冲区取出数据；如果缓冲区已满并且sendq队列不为空，优先读取缓冲区头部的数据，并将队列的G的数据写入缓冲区尾部；

无缓冲区：将当前goroutine加入recvq队列，等到写goroutine的唤醒

### 易错点

1. channel未初始化，写入或者读取都会阻塞
2. 往close的channel写入数据会发生panic
3. close未初始化channel会发生panic
4. close已经close过的channel会发生panic

## 08 SingleFlight

### 基本概念

SingleFlight 是 Go 开发组提供的一个扩展并发原语。它的作用是，在处理多个 goroutine 同时调用同一个函数的时候，只让一个 goroutine 去调用这个函数，等到这个 goroutine 返回结果的时候，再把结果返回给这几个同时调用的 goroutine，这样可以减少并发调用的数量。

### 与sync.Once的区别

1. sync.Once 不是只在并发的时候保证只有一个 goroutine 执行函数 f，而是会保证永远只执行一次，而 SingleFlight 是每次调用都重新执行，并且在多个请求同时调用的时候只有一个执行。
2. sync.Once 主要是用在单次初始化场景中，而 SingleFlight 主要用在合并并发请求的场景中

### 应用场景

使用 SingleFlight 时，可以通过合并请求的方式降低对下游服务的并发压力，从而提高系统的性能，常常用于缓存系统中

### 基本方法

1. `func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool)`

提供一个 key，对于同一个 key，在同一时间只有一个在执行，同一个 key 并发的请求会等待。第一个执行的请求返回的结果，就是它的返回结果。函数 fn 是一个无参的函数，返回一个结果或者 error，而 Do 方法会返回函数执行的结果或者是 error，shared 会指示 v 是否返回给多个请求

1. `func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result`

类似 Do 方法，只不过是返回一个 chan，等 fn 函数执行完，产生了结果以后，就能从这个 chan 中接收这个结果

1. `func (g *Group) Forget(key string)`

告诉 Group 忘记这个 key。这样一来，之后这个 key 请求会执行 f，而不是等待前一个未完成的 fn 函数的结果

### 实现方法

SingleFlight 定义一个辅助对象 call，这个 call 就代表正在执行 fn 函数的请求或者是已经执行完的请求

在Do方法中，传入key与执行函数，加锁，查询是否存在key，如果存在，等待第一个请求完成并返回。如果不存在，创建一个call，将这个call加入到key map中，释放锁，执行doCall函数，执行完实际函数后，删除key。

```
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
  g.mu.Lock()
  if g.m == nil {
    g.m = make(map[string]*call)
  }
  if c, ok := g.m[key]; ok {//如果已经存在相同的key
    c.dups++
    g.mu.Unlock()
    c.wg.Wait() //等待这个key的第一个请求完成
    return c.val, c.err, true //使用第一个key的请求结果
  }
  c := new(call) // 第一个请求，创建一个call
  c.wg.Add(1)
  g.m[key] = c //加入到key map中
  g.mu.Unlock()


  g.doCall(c, key, fn) // 调用方法
  return c.val, c.err, c.dups > 0
}

func (g *Group) doCall(c *call, key string, fn func() (interface{}, error)) {
  c.val, c.err = fn()
  c.wg.Done()


  g.mu.Lock()
  if !c.forgotten { // 已调用完，删除这个key
    delete(g.m, key)
  }
  for _, ch := range c.chans {
    ch <- Result{c.val, c.err, c.dups > 0}
  }
  g.mu.Unlock()
}
```





