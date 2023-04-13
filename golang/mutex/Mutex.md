# Data Race
### 定义&&举例
Go语言天生支持高并发，但是并发访问有时候会出现各种问题。
比如多个 goroutine 并发更新同一个资源，像计数器；同时更新用户的账户信息；秒杀系统；往同一个 buffer 中并发写入数据等等。如果没有互斥控制，就会出现一些异常情况，比如计数器的计数不准确、用户的账户可能出现透支、秒杀系统出现超卖、buffer 中的数据混乱，都会造成数据不准确。
```go
package main

import (
        "fmt"
        "sync"
    )

func main() {
    var count = 0
    // 使用WaitGroup等待10个goroutine完成
	var wg sync.WaitGroup
    wg.Add(10)
    for i := 0; i < 10; i++ {
        go func() {
            defer wg.Done()
            // 对变量count执行10次加1
            for j := 0; j < 100000; j++ {
                 count++
             }
        }()
    }
    // 等待10个goroutine完成
    wg.Wait()
    fmt.Println(count)
}
```
sync.WaitGroup是用来控制等待一组goroutine全部执行完成任务
![image.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1663458833623-dd672b39-8557-48e9-aced-6b2fd30b8372.png#clientId=u556fb619-db69-4&errorMessage=unknown%20error&from=paste&height=396&id=u7bdaf2e6&name=image.png&originHeight=792&originWidth=1534&originalType=binary&ratio=1&rotation=0&showTitle=false&size=105728&status=error&style=none&taskId=ufe4a9e61-a682-4a34-b228-482d48c3a62&title=&width=767)
这是因为，count++ 不是一个原子操作，它至少包含几个步骤，比如读取变量 count 的当前值，对这个值加 1，把结果再保存到 count 中。因为不是原子操作，就可能有并发的问题。
比如，10 个 goroutine 同时读取到 count 的值为 9527，接着各自按照自己的逻辑加 1，值变成了 9528，然后把这个结果再写回到 count 变量。但是，实际上，此时我们增加的总数应该是 10 才对，这里却只增加了 1，好多计数都被“吞”掉了。这是并发访问共享数据的常见错误。
针对这个问题，Go 提供了一个检测并发访问共享资源是否有问题的工具： race detector，它可以帮助我们自动发现程序有没有 data race 的问题。Go race detector 是基于 Google 的 C/C++ [sanitizers](https://github.com/google/sanitizers) 技术实现的，编译器通过探测所有的内存访问，加入代码能监视对这些内存地址的访问（读还是写）。在代码运行的时候，race detector 就能监控到对共享变量的非同步访问，出现 race 的时候，就会打印出警告信息。
在编译（compile）、测试（test）或者运行（run）Go 代码的时候，加上 race 参数，就有可能发现并发问题。使用 go run -race main.go，就会输出警告信息。
![图片.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1663557582066-6c9e3a5f-f7ca-421d-9a64-8b28a5470206.png#clientId=ud10962bb-2c31-4&errorMessage=unknown%20error&from=paste&height=565&id=uaec5d786&name=%E5%9B%BE%E7%89%87.png&originHeight=565&originWidth=661&originalType=binary&ratio=1&rotation=0&showTitle=false&size=77203&status=error&style=none&taskId=u9bbaf2a1-545c-4034-af9a-17468b57369&title=&width=661)
把开启了 race 的程序部署在线上，还是比较影响性能的。运行 go tool compile -race -S main.go，可以查看计数器例子的代码，重点关注一下 count++ 前后的编译后的代码
![图片.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1663472396143-2cacba1d-0eef-4c12-a29d-61a08d80b737.png#clientId=u9ae323bf-0626-4&errorMessage=unknown%20error&from=paste&height=677&id=u55b91dba&name=%E5%9B%BE%E7%89%87.png&originHeight=1354&originWidth=2304&originalType=binary&ratio=1&rotation=0&showTitle=false&size=881476&status=error&style=none&taskId=ua5e5d578-51e8-44e1-a745-5388fcd65c2&title=&width=1152)
在编译的代码中，增加了 runtime.racefuncenter、runtime.raceread、runtime.racewrite、runtime.racefuncexit 等检测 data race 的方法。通过这些插入的指令，Go race detector 工具就能够成功地检测出 data race 问题了。
### Go官方介绍

   - [race detector](https://go.dev/blog/race-detector)
   - [examples go data race](https://go.dev/doc/articles/race_detector#Introduction)
# Mutex
互斥锁是并发控制的一个基本手段，是为了避免竞争而建立的一种并发控制机制。
## 临界区
在并发编程中，如果程序中的一部分会被并发访问或修改，那么，为了避免并发访问导致的意想不到的结果，这部分程序需要被保护起来，这部分被保护起来的程序，就叫做临界区。可以说，临界区就是一个被共享的资源，或者说是一个整体的一组共享资源，比如对数据库的访问、对某一个共享数据结构的操作、对一个 I/O 设备的使用、对一个连接池中的连接的调用，等等。如果很多线程同步访问临界区，就会造成访问或操作错误，这当然不是我们希望看到的结果。所以，我们可以使用互斥锁，限定临界区只能同时由一个线程持有。当临界区由一个线程持有的时候，其它线程如果想进入这个临界区，就会返回失败，或者是等待。直到持有的线程退出临界区，这些等待线程中的某一个才有机会接着持有这个临界区。
![图片.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1663473056604-74c7ae24-94bc-46be-a40d-3bd4e2d82976.png#clientId=u9ae323bf-0626-4&errorMessage=unknown%20error&from=paste&height=986&id=u20871f8d&name=%E5%9B%BE%E7%89%87.png&originHeight=1972&originWidth=3747&originalType=binary&ratio=1&rotation=0&showTitle=false&size=439531&status=error&style=none&taskId=u4e5ff15d-ef03-4527-aca3-692d08c2efb&title=&width=1873.5)	互斥锁也可叫做排他锁（[understanding real-world concurrency bugs in go](https://songlh.github.io/paper/go-study.pdf)）
## 基本使用
```go
type Locker interface {
    Lock()
    Unlock()
}

func(m *Mutex)Lock()
func(m *Mutex)Unlock()
```
当一个 goroutine 通过调用 Lock 方法获得了这个锁的拥有权后， 其它请求锁的 goroutine 就会阻塞在 Lock 方法的调用上，直到锁被释放并且自己获取到了这个锁的拥有权。
```go
package main

import (
        "fmt"
        "sync"
    )
    
func main() {
    var mutex sync.Mutex
	var count = 0
	// 使用WaitGroup等待10个goroutine完成
	var wg sync.WaitGroup
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			// 对变量count执行10次加1
			for j := 0; j < 100000; j++ {
				mutex.Lock()
				count++
				mutex.Unlock()
			}
		}()
	}
	// 等待10个goroutine完成
	wg.Wait()
	return count
}

```
## 组合使用
```go
func main() {
    var counter Counter
    var wg sync.WaitGroup
    wg.Add(10)
    for i := 0; i < 10; i++ {
        go func() {
            defer wg.Done()
            for j := 0; j < 100000; j++ {
                counter.Lock()
                counter.Count++
                counter.Unlock()
            }
        }()
    }
    wg.Wait()
    fmt.Println(counter.Count)
}


type Counter struct {
    sync.Mutex
    Count uint64
}
```
```go
func main() {
    // 封装好的计数器
    var counter Counter

    var wg sync.WaitGroup
    wg.Add(10)

    // 启动10个goroutine
    for i := 0; i < 10; i++ {
        go func() {
            defer wg.Done()
            // 执行10万次累加
            for j := 0; j < 100000; j++ {
                counter.Incr() // 受到锁保护的方法
            }
        }()
    }
    wg.Wait()
    fmt.Println(counter.Count())
}

// 线程安全的计数器类型
type Counter struct {
    CounterType int
    Name        string

    mu    sync.Mutex
    count uint64
}

// 加1的方法，内部使用互斥锁保护
func (c *Counter) Incr() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}

// 得到计数器的值，也需要锁保护
func (c *Counter) Count() uint64 {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```
## 源码实现
### 初版
通过一个 flag 变量，标记当前的锁是否被某个 goroutine 持有。如果这个 flag 的值是 1，就代表锁已经被持有，那么，其它竞争的 goroutine 只能等待；如果这个 flag 的值是 0，就可以通过 CAS（compare-and-swap，或者 compare-and-set）将这个 flag 设置为 1，标识锁被当前的这个 goroutine 持有了。
```go
   // CAS操作，当时还没有抽象出atomic包
    func cas(val *int32, old, new int32) bool
    func semacquire(*int32)
    func semrelease(*int32)
    // 互斥锁的结构，包含两个字段
    type Mutex struct {
        key  int32 // 锁是否被持有的标识
        sema int32 // 信号量专用，用以阻塞/唤醒goroutine
    }
    
    // 保证成功在val上增加delta的值
    func xadd(val *int32, delta int32) (new int32) {
        for {
            v := *val
            if cas(val, v, v+delta) {
                return v + delta
            }
        }
        panic("unreached")
    }
    
    // 请求锁
    func (m *Mutex) Lock() {
        if xadd(&m.key, 1) == 1 { //标识加1，如果等于1，成功获取到锁
            return
        }
        semacquire(&m.sema) // 否则阻塞等待
    }
    // 释放锁
    func (m *Mutex) Unlock() {
        if xadd(&m.key, -1) == 0 { // 将标识减去1，如果等于0，则没有其它等待者
          return
        }
        semrelease(&m.sema) // 唤醒其它阻塞的goroutine
    }    
```
CAS(compare and swap) 指令将给定的值和一个内存地址中的值进行比较，如果它们是同一个值，就使用新值替换内存地址中的值，这个操作是原子性的，原子性保证这个指令总是基于最新的值进行计算，如果同时有其它线程已经修改了这个值，那么，CAS 会返回失败。
CAS 是实现互斥锁和同步原语的基础。
#### 字段含义

1. key：是一个 flag，用来标识这个排外锁是否被某个 goroutine 所持有，如果 key 大于等于 1，说明这个排外锁已经被持有；
2. sema：是个信号量变量，用来控制等待 goroutine 的阻塞休眠和唤醒。

![图片.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1663488895943-0d983884-6eab-444e-af93-1cc93c31ae5e.png#clientId=u9ae323bf-0626-4&errorMessage=unknown%20error&from=paste&height=1100&id=u4f166d35&name=%E5%9B%BE%E7%89%87.png&originHeight=2199&originWidth=3032&originalType=binary&ratio=1&rotation=0&showTitle=false&size=547872&status=error&style=none&taskId=u5652af9a-c803-4ed7-b09a-d18c48dba61&title=&width=1516)	调用 Lock 请求锁的时候，通过 xadd 方法进行 CAS 操作，xadd 方法通过循环执行 CAS 操作直到成功，保证对 key 加 1 的操作成功完成。如果比较幸运，锁没有被别的 goroutine 持有，那么，Lock 方法成功地将 key 设置为 1，这个 goroutine 就持有了这个锁；如果锁已经被别的 goroutine 持有了，那么，当前的 goroutine 会把 key 加 1，而且还会调用 semacquire 方法，使用信号量将自己休眠，等锁释放的时候，信号量会将它唤醒。
持有锁的 goroutine 调用 Unlock 释放锁时，它会将 key 减 1。如果当前没有其它等待这个锁的 goroutine，这个方法就返回了。但是，如果还有等待此锁的其它 goroutine，那么，它会调用 semrelease 方法，利用信号量唤醒等待锁的其它 goroutine 中的一个。
初版的 Mutex 利用 CAS 原子操作，对 key 这个标志量进行设置。key 不仅仅标识了锁是否被 goroutine 所持有，还记录了当前持有和等待获取锁的 goroutine 的数量。
#### 谁申请谁释放
Unlock 方法可以被任意的 goroutine 调用释放锁，即使是没持有这个互斥锁的 goroutine，也可以进行这个操作。这是因为，Mutex 本身并没有包含持有这把锁的 goroutine 的信息，所以，Unlock 也不会对此进行检查。Mutex 的这个设计一直保持至今。
必须保证goroutine尽可能不去释放自己未持有的锁。
```go
type Foo struct {
    mu    sync.Mutex
    count int
}

func (f *Foo) Bar() {
    f.mu.Lock()

    if f.count < 1000 {
        f.count += 3
        f.mu.Unlock() // 此处释放锁
        return
    }

    f.count++
    f.mu.Unlock() // 此处释放锁
    return
}
```
#### 可重入锁
当一个线程获取锁时，如果没有其它线程拥有这个锁，那么，这个线程就成功获取到这个锁。之后，如果其它线程再请求这个锁，就会处于阻塞等待的状态。但是，如果拥有这把锁的线程再请求这把锁的话，不会阻塞，而是成功返回，所以叫可重入锁（有时候也叫做递归锁）。只要你拥有这把锁，你可以可着劲儿地调用，比如通过递归实现一些算法，调用者不会阻塞或者死锁。
```go

func foo(l sync.Locker) {
    fmt.Println("in foo")
    l.Lock()
    bar(l)
    l.Unlock()
}


func bar(l sync.Locker) {
    l.Lock()
    fmt.Println("in bar")
    l.Unlock()
}


func main() {
    l := &sync.Mutex{}
    foo(l)
}
```
![图片.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1663829960830-a54aece6-8ad5-42ac-8f3f-404a70872936.png#clientId=uf5d7a11e-457f-4&from=paste&height=406&id=ue514d8bf&name=%E5%9B%BE%E7%89%87.png&originHeight=406&originWidth=614&originalType=binary&ratio=1&rotation=0&showTitle=false&size=69046&status=done&style=none&taskId=u31e81ef6-9270-4a8f-b9a6-504eb08c87d&title=&width=614)
#### [Goroutine Id](https://chai2010.cn/advanced-go-programming-book/ch3-asm/ch3-08-goroutine-id.html#38-%E4%BE%8B%E5%AD%90goroutine-id)
```go
package main
import (
	"fmt"
	"runtime"
	"strconv"
	"strings"
	"sync"
)
func GoID() int {
	var buf [64]byte
	n := runtime.Stack(buf[:], false)
	idField := strings.Fields(strings.TrimPrefix(string(buf[:n]), "goroutine "))[0]
	id, err := strconv.Atoi(idField)
	if err != nil {
		panic(fmt.Sprintf("cannot get goroutine id: %v", err))
	}
	return id
}
func main() {
	fmt.Println("main", GoID())
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		i := i
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Println(i, GoID())
		}()
	}
	wg.Wait()
}
```
通过 runtime.Stack 方法获取栈帧信息，栈帧信息里包含 goroutine id
获取运行时的 g 指针，反解出对应的 g 的结构。每个运行的 goroutine 结构的 g 指针保存在当前 goroutine 的一个叫做 TLS 对象中。

   - 第一步：我们先获取到 TLS 对象；
   - 第二步：再从 TLS 中获取 goroutine 结构的 g 指针；
   - 第三步：再从 g 指针中取出 goroutine id。

不同go语言版本的goroutine的结构可能不一样，需要根据go的[不同版本](https://github.com/golang/go/blob/89f687d6dbc11613f715d1644b4983905293dd33/src/runtime/runtime2.go#L412)进行调整获取
获取go不同版本的goroutine id：[petermattis/goid](https://github.com/petermattis/goid)
#### 可重入锁实现
```go
// RecursiveMutex 包装一个Mutex,实现可重入
type RecursiveMutex struct {
    sync.Mutex
    owner     int64 // 当前持有锁的goroutine id
    recursion int32 // 这个goroutine 重入的次数
}

func (m *RecursiveMutex) Lock() {
    gid := goid.Get()
    // 如果当前持有锁的goroutine就是这次调用的goroutine,说明是重入
    if atomic.LoadInt64(&m.owner) == gid {
        m.recursion++
        return
    }
    m.Mutex.Lock()
    // 获得锁的goroutine第一次调用，记录下它的goroutine id,调用次数加1
    atomic.StoreInt64(&m.owner, gid)
    m.recursion = 1
}

func (m *RecursiveMutex) Unlock() {
    gid := goid.Get()
    // 非持有锁的goroutine尝试释放锁，错误的使用
    if atomic.LoadInt64(&m.owner) != gid {
        panic(fmt.Sprintf("wrong the owner(%d): %d!", m.owner, gid))
    }
    // 调用次数减1
    m.recursion--
    if m.recursion != 0 { // 如果这个goroutine还没有完全释放，则直接返回
        return
    }
    // 此goroutine最后一次调用，需要释放锁
    atomic.StoreInt64(&m.owner, -1)
    m.Mutex.Unlock()
}
```
```go

// Token方式的递归锁
type TokenRecursiveMutex struct {
    sync.Mutex
    token     int64
    recursion int32
}

// 请求锁，需要传入token
func (m *TokenRecursiveMutex) Lock(token int64) {
    if atomic.LoadInt64(&m.token) == token { //如果传入的token和持有锁的token一致，说明是递归调用
        m.recursion++
        return
    }
    m.Mutex.Lock() // 传入的token不一致，说明不是递归调用
    // 抢到锁之后记录这个token
    atomic.StoreInt64(&m.token, token)
    m.recursion = 1
}

// 释放锁
func (m *TokenRecursiveMutex) Unlock(token int64) {
    if atomic.LoadInt64(&m.token) != token { // 释放其它token持有的锁
        panic(fmt.Sprintf("wrong the owner(%d): %d!", m.token, token))
    }
    m.recursion-- // 当前持有这个锁的token释放锁
    if m.recursion != 0 { // 还没有回退到最初的递归调用
        return
    }
    atomic.StoreInt64(&m.token, 0) // 没有递归调用了，释放锁
    m.Mutex.Unlock()
}

```
go语言从1.14版本起，对defer做了优化，采用更加有效的内连方式，取代之前的生成 defer 对象到 defer chain 中，defer 对耗时的影响微乎其微了。
```go
func (f *Foo) Bar() {
    f.mu.Lock()
    defer f.mu.Unlock()


    if f.count < 1000 {
        f.count += 3
        return
    }


    f.count++
    return
}
```
如果临界区只是方法的一部分，为了尽快释放锁，还是应该第一时间调用Unlock，而不是一直等到方法返回时再释放。
初版的 Mutex 实现之后，Go 开发组又对 Mutex 做了一些微调，比如把字段类型变成了 uint32 类型；调用 Unlock 方法会做检查；使用 atomic 包的同步原语执行原子操作等等。
但是，初版的 Mutex 实现有一个问题：请求锁的 goroutine 会排队等待获取互斥锁。虽然这貌似很公平，但是从性能上来看，却不是最优的。因为如果我们能够把锁交给正在占用 CPU 时间片的 goroutine 的话，那就不需要做上下文的切换，在高并发的情况下，可能会有更好的性能。
### 给新人机会
Go 开发者在 2011 年 6 月 30 日的 commit 中对 Mutex 做了一次大的调整，调整后的 Mutex 实现如下：
```go
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
![图片.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1663490299143-81c4462e-4af0-4524-be71-f9f0f342a749.png#clientId=u005fdc75-1856-4&errorMessage=unknown%20error&from=paste&height=878&id=u57e7f459&name=%E5%9B%BE%E7%89%87.png&originHeight=1756&originWidth=2978&originalType=binary&ratio=1&rotation=0&showTitle=false&size=409133&status=error&style=none&taskId=uc16dc5eb-e7e7-4654-be7d-61ec07a809a&title=&width=1489)	state 是一个复合型的字段，一个字段包含多个意义，这样可以通过尽可能少的内存来实现互斥锁。这个字段的第一位（最小的一位）来表示这个锁是否被持有，第二位代表是否有唤醒的 goroutine，剩余的位数代表的是等待此锁的 goroutine 数。所以，state 这一个字段被分成了三部分，代表三个数据。
```go
   func (m *Mutex) Lock() {
        // Fast path: 幸运case，能够直接获取到锁
        if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
            return
        }

        awoke := false
        for {
            old := m.state
            new := old | mutexLocked // 新状态加锁
            if old&mutexLocked != 0 {
                new = old + 1<<mutexWaiterShift //等待者数量加一
            }
            if awoke {
                // goroutine是被唤醒的，
                // 新状态清除唤醒标志
                new &^= mutexWoken
            }
            if atomic.CompareAndSwapInt32(&m.state, old, new) {//设置新状态
                if old&mutexLocked == 0 { // 锁原状态未加锁
                    break
                }
                runtime.Semacquire(&m.sema) // 请求信号量
                awoke = true
            }
        }
    }
```
首先是通过 CAS 检测 state 字段中的标志（第 3 行），如果没有 goroutine 持有锁，也没有等待持有锁的 gorutine，那么，当前的 goroutine 就很幸运，可以直接获得锁，这也是注释中的 Fast path 的意思。
如果不够幸运，state 不是零值，那么就通过一个循环进行检查。
我们先前知道，如果想要获取锁的 goroutine 没有机会获取到锁，就会进行休眠，但是在锁释放唤醒之后，它并不能像先前一样直接获取到锁，还是要和正在请求锁的 goroutine 进行竞争。这会给后来请求锁的 goroutine 一个机会，也让 CPU 中正在执行的 goroutine 有更多的机会获取到锁，在一定程度上提高了程序的性能。
for 循环是不断尝试获取锁，如果获取不到，就通过 runtime.Semacquire(&m.sema) 休眠，休眠醒来之后 awoke 置为 true，尝试争抢锁。
代码中的第 10 行将当前的 flag 设置为加锁状态，如果能成功地通过 CAS 把这个新值赋予 state（第 19 行和第 20 行），就代表抢夺锁的操作成功了。
不过，需要注意的是，如果成功地设置了 state 的值，但是之前的 state 是有锁的状态，那么，state 只是清除 mutexWoken 标志或者增加一个 waiter 而已。
请求锁的 goroutine 有两类，一类是新来请求锁的 goroutine，另一类是被唤醒的等待请求锁的 goroutine。锁的状态也有两种：加锁和未加锁。
![图片.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1663490405580-40a0258a-7bf3-4879-9cf2-34aa7dd92f8c.png#clientId=u005fdc75-1856-4&errorMessage=unknown%20error&from=paste&height=310&id=ue2f290a4&name=%E5%9B%BE%E7%89%87.png&originHeight=620&originWidth=1690&originalType=binary&ratio=1&rotation=0&showTitle=false&size=464434&status=error&style=none&taskId=u1cd09e54-c9fd-4dfd-9bc3-4fe4d794c7a&title=&width=845)
```go

   func (m *Mutex) Unlock() {
        // Fast path: drop lock bit.
        new := atomic.AddInt32(&m.state, -mutexLocked) //去掉锁标志
        if (new+mutexLocked)&mutexLocked == 0 { //本来就没有加锁
            panic("sync: unlock of unlocked mutex")
        }
    
        old := new
        for {
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 { // 没有等待者，或者有唤醒的waiter，或者锁原来已加锁
                return
            }
            new = (old - 1<<mutexWaiterShift) | mutexWoken // 新状态，准备唤醒goroutine，并设置唤醒标志
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime.Semrelease(&m.sema)
                return
            }
            old = m.state
        }
    }
```
第 3 行是尝试将持有锁的标识设置为未加锁的状态，这是通过减 1 而不是将标志位置零的方式实现。第 4 到 6 行还会检测原来锁的状态是否已经未加锁的状态，如果是 Unlock 一个未加锁的 Mutex 会直接 panic。
不过，即使将加锁置为未加锁的状态，这个方法也不能直接返回，还需要一些额外的操作，因为还可能有一些等待这个锁的 goroutine（有时候我也把它们称之为 waiter）需要通过信号量的方式唤醒它们中的一个。所以接下来的逻辑有两种情况。
第一种情况，如果没有其它的 waiter，说明对这个锁的竞争的 goroutine 只有一个，那就可以直接返回了；如果这个时候有唤醒的 goroutine，或者是又被别人加了锁，那么，无需我们操劳，其它 goroutine 自己干得都很好，当前的这个 goroutine 就可以放心返回了。
第二种情况，如果有等待者，并且没有唤醒的 waiter，那就需要唤醒一个等待的 waiter。在唤醒之前，需要将 waiter 数量减 1，并且将 mutexWoken 标志设置上，这样，Unlock 就可以返回了。
通过这样复杂的检查、判断和设置，我们就可以安全地将一把互斥锁释放了。
**相对于初版的设计，这次的改动主要就是，新来的 goroutine 也有机会先获取到锁，甚至一个 goroutine 可能连续获取到锁，打破了先来先得的逻辑。但是，代码复杂度也显而易见。虽然这一版的 Mutex 已经给新来请求锁的 goroutine 一些机会，让它参与竞争，没有空闲的锁或者竞争失败才加入到等待队列中。**
### 多给些机会
在 2015 年 2 月的改动中，如果新来的 goroutine 或者是被唤醒的 goroutine 首次获取不到锁，它们就会通过自旋（spin，通过循环不断尝试，spin 的逻辑是在[runtime ](https://github.com/golang/go/blob/846dce9d05f19a1f53465e62a304dea21b99f910/src/runtime/proc.go#L5580)实现的）的方式，尝试检查锁是否被释放。在尝试一定的自旋次数后，再执行原来的逻辑。
```go
   func (m *Mutex) Lock() {
        // Fast path: 幸运之路，正好获取到锁
        if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
            return
        }

        awoke := false
        iter := 0
        for { // 不管是新来的请求锁的goroutine, 还是被唤醒的goroutine，都不断尝试请求锁
            old := m.state // 先保存当前锁的状态
            new := old | mutexLocked // 新状态设置加锁标志
            if old&mutexLocked != 0 { // 锁还没被释放
                if runtime_canSpin(iter) { // 还可以自旋
                    if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                        atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                        awoke = true
                    }
                    runtime_doSpin()
                    iter++
                    continue // 自旋，再次尝试请求锁
                }
                new = old + 1<<mutexWaiterShift
            }
            if awoke { // 唤醒状态
                if new&mutexWoken == 0 {
                    panic("sync: inconsistent mutex state")
                }
                new &^= mutexWoken // 新状态清除唤醒标记
            }
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                if old&mutexLocked == 0 { // 旧状态锁已释放，新状态成功持有了锁，直接返回
                    break
                }
                runtime_Semacquire(&m.sema) // 阻塞等待
                awoke = true // 被唤醒
                iter = 0
            }
        }
    }
```
如果可以 spin 的话，第 9 行的 for 循环会重新检查锁是否释放。对于临界区代码执行非常短的场景来说，这是一个非常好的优化。因为临界区的代码耗时很短，锁很快就能释放，而抢夺锁的 goroutine 不用通过休眠唤醒方式等待调度，直接 spin 几次，可能就获得了锁。
### 解决饥饿
因为新来的 goroutine 也参与竞争，有可能每次都会被新来的 goroutine 抢到获取锁的机会，在极端情况下，等待中的 goroutine 可能会一直获取不到锁，这就是饥饿问题。
先前版本的 Mutex 遇到的也是同样的困境，“悲惨”的 goroutine 总是得不到锁。Mutex 不能容忍这种事情发生。所以，2016 年 Go 1.9 中 Mutex 增加了饥饿模式，让锁变得更公平，不公平的等待时间限制在 1 毫秒，并且修复了一个大 Bug：总是把唤醒的 goroutine 放在等待队列的尾部，会导致更加不公平的等待时间。
之后，2018 年，Go 开发者将 fast path 和 slow path 拆成独立的方法，以便内联，提高性能。2019 年也有一个 Mutex 的优化，虽然没有对 Mutex 做修改，但是，对于 Mutex 唤醒后持有锁的那个 waiter，调度器可以有更高的优先级去执行，这已经是很细致的性能优化了。
![图片.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1663492500474-a78b2f8f-bcc5-4945-add4-b60d656b3fdc.png#clientId=u005fdc75-1856-4&errorMessage=unknown%20error&from=paste&height=822&id=ue25bc436&name=%E5%9B%BE%E7%89%87.png&originHeight=1644&originWidth=3096&originalType=binary&ratio=1&rotation=0&showTitle=false&size=462428&status=error&style=none&taskId=ue0539352-e6c8-4604-a07d-52dd5cbf3bb&title=&width=1548)
```go
   type Mutex struct {
        state int32
        sema  uint32
    }
    
    const (
        mutexLocked = 1 << iota // mutex is locked
        mutexWoken // 代表唤醒
        mutexStarving // 从state字段中分出一个饥饿标记
        mutexWaiterShift = iota // 代表位移
    
        starvationThresholdNs = 1e6
    )
```
```go
func (m *Mutex) Lock() {
    // 如果mutext的state没有被锁，也没有等待/唤醒的goroutine, 锁处于正常状态，那么获得锁，返回.
    // 比如锁第一次被goroutine请求时，就是这种状态。或者锁处于空闲的时候，也是这种状态。
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }
    // 标记本goroutine的等待时间
    var waitStartTime int64
    // 本goroutine是否已经处于饥饿状态
    starving := false
    // 本goroutine是否已唤醒
    awoke := false
    
    // 自旋次数
    iter := 0
    
    // 复制锁的当前状态
    old := m.state
    
    for {
        // 第一个条件是state已被锁，但是不是饥饿状态。如果是饥饿状态，自旋时没有用的，锁的拥有权直接交给了等待队列的第一个。
        // 第二个条件是还可以自旋，多核、压力不大并且在一定次数内可以自旋， 具体的条件可以参考`sync_runtime_canSpin`的实现。
        // 如果满足这两个条件，不断自旋来等待锁被释放、或者进入饥饿状态、或者不能再自旋。
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 自旋的过程中如果发现state还没有设置woken标识，则设置它的woken标识， 并标记自己为被唤醒。
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
        
        // 到了这一步， state的状态可能是：
        // 1. 锁还没有被释放，锁处于正常状态
        // 2. 锁还没有被释放， 锁处于饥饿状态
        // 3. 锁已经被释放， 锁处于正常状态
        // 4. 锁已经被释放， 锁处于饥饿状态
        //
        // 并且本gorutine的 awoke可能是true, 也可能是false (其它goutine已经设置了state的woken标识)
        // new 复制 state的当前状态， 用来设置新的状态
        // old 是锁当前的状态
        new := old
        
        // 如果old state状态不是饥饿状态, new state 设置锁， 尝试通过CAS获取锁,
        // 如果old state状态是饥饿状态, 则不设置new state的锁，因为饥饿状态下锁直接转给等待队列的第一个.
        if old&mutexStarving == 0 {
            new |= mutexLocked
        }
        // 将等待队列的等待者的数量加1
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }
        
        // 如果当前goroutine已经处于饥饿状态， 并且old state的已被加锁,
        // 将new state的状态标记为饥饿状态, 将锁转变为饥饿状态.
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }
        
        // 如果本goroutine已经设置为唤醒状态, 需要清除new state的唤醒标记, 因为本goroutine要么获得了锁，要么进入休眠，
        // 总之state的新状态不再是woken状态.
        if awoke {
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }
        
        // 通过CAS设置new state值.
        // 注意new的锁标记不一定是true, 也可能只是标记一下锁的state是饥饿状态.
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // 如果old state的状态是未被锁状态，并且锁不处于饥饿状态,
            // 那么当前goroutine已经获取了锁的拥有权，返回
            if old&(mutexLocked|mutexStarving) == 0 {
                break 
            }
            
            // 设置/计算本goroutine的等待时间
            queueLifo := waitStartTime != 0
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            
            // 既然未能获取到锁， 那么就使用sleep原语阻塞本goroutine
            // 如果是新来的goroutine,queueLifo=false, 加入到等待队列的尾部，耐心等待
            // 如果是唤醒的goroutine, queueLifo=true, 加入到等待队列的头部
            runtime_SemacquireMutex(&m.sema, queueLifo)
            
            // sleep之后，此goroutine被唤醒
            // 计算当前goroutine是否已经处于饥饿状态.
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            // 得到当前的锁状态
            old = m.state
            
            // 如果当前的state已经是饥饿状态
            // 那么锁应该处于Unlock状态，那么应该是锁被直接交给了本goroutine
            if old&mutexStarving != 0 { 
                
                // 如果当前的state已被锁，或者已标记为唤醒， 或者等待的队列中不为空,
                // 那么state是一个非法状态
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
                
                // 当前goroutine用来设置锁，并将等待的goroutine数减1.
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                
                // 如果本goroutine是最后一个等待者，或者它并不处于饥饿状态，
                // 那么我们需要把锁的state状态设置为正常模式.
                if !starving || old>>mutexWaiterShift == 1 {
                    // 退出饥饿模式
                    delta -= mutexStarving
                }
                
                // 设置新state, 因为已经获得了锁，退出、返回
                atomic.AddInt32(&m.state, delta)
                break
            }
            
            // 如果当前的锁是正常模式，本goroutine被唤醒，自旋次数清零，从for循环开始处重新开始
            awoke = true
            iter = 0
        } else { // 如果CAS不成功，重新获取锁的state, 从for循环开始处重新开始
            old = m.state
        }
    }
}
```

跟之前的实现相比，当前的 Mutex 最重要的变化，就是增加饥饿模式。第 12 行将饥饿模式的最大等待时间阈值设置成了 1 毫秒，这就意味着，一旦等待者等待的时间超过了这个阈值，Mutex 的处理就有可能进入饥饿模式，优先让等待者先获取到锁，新来的同学主动谦让一下，给老同志一些机会。
通过加入饥饿模式，可以避免把机会全都留给新来的 goroutine，保证了请求锁的 goroutine 获取锁的公平性，对于我们使用锁的业务代码来说，不会有业务一直等待锁不被处理。
```go
func (m *Mutex) Unlock() {
    // 如果state不是处于锁的状态, 那么就是Unlock根本没有加锁的mutex, panic
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if (new+mutexLocked)&mutexLocked == 0 {
        throw("sync: unlock of unlocked mutex")
    }
    
    // 释放了锁，还得需要通知其它等待者
    // 锁如果处于饥饿状态，直接交给等待队列的第一个, 唤醒它，让它去获取锁
    // 锁如果处于正常状态，
    // new state如果是正常状态
    if new&mutexStarving == 0 {
        old := new
        for {
            // 如果没有等待的goroutine, 或者锁不处于空闲的状态，直接返回.
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }
            // 将等待的goroutine数减一，并设置woken标识
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            // 设置新的state, 这里通过信号量会唤醒一个阻塞的goroutine去获取锁.
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime_Semrelease(&m.sema, false)
                return
            }
            old = m.state
        }
    } else {
        // 饥饿模式下， 直接将锁的拥有权传给等待队列中的第一个.
        // 注意此时state的mutexLocked还没有加锁，唤醒的goroutine会设置它。
        // 在此期间，如果有新的goroutine来请求锁， 因为mutex处于饥饿状态， mutex还是被认为处于锁状态，
        // 新来的goroutine不会把锁抢过去.
        runtime_Semrelease(&m.sema, true)
    }
}
```
#### 饥饿模式与正常模式
请求锁时调用的 Lock 方法中一开始是 fast path，这是一个幸运的场景，当前的 goroutine 幸运地获得了锁，没有竞争，直接返回，否则就进入了 lockSlow 方法。这样的设计，方便编译器对 Lock 方法进行内联，你也可以在程序开发中应用这个技巧。
正常模式下，waiter 都是进入先入先出队列，被唤醒的 waiter 并不会直接持有锁，而是要和新来的 goroutine 进行竞争。新来的 goroutine 有先天的优势，它们正在 CPU 中运行，可能它们的数量还不少，所以，在高并发情况下，被唤醒的 waiter 可能比较悲剧地获取不到锁，这时，它会被插入到队列的前面。如果 waiter 获取不到锁的时间超过阈值 1 毫秒，那么，这个 Mutex 就进入到了饥饿模式。
在饥饿模式下，Mutex 的拥有者将直接把锁交给队列最前面的 waiter。新来的 goroutine 不会尝试获取锁，即使看起来锁没有被持有，它也不会去抢，也不会 spin，它会乖乖地加入到等待队列的尾部。
如果拥有 Mutex 的 waiter 发现下面两种情况的其中之一，它就会把这个 Mutex 转换成正常模式:

   - 此 waiter 已经是队列中的最后一个 waiter 了，没有其它的等待锁的 goroutine 了；
   - 此 waiter 的等待时间小于 1 毫秒。

正常模式拥有更好的性能，因为即使有等待抢锁的 waiter，goroutine 也可以连续多次获取到锁。
饥饿模式是对公平性和性能的一种平衡，它避免了某些 goroutine 长时间的等待锁。在饥饿模式下，优先对待的是那些一直在等待的 waiter。
## 总结：
### Lock
### ![](https://cdn.nlark.com/yuque/0/2022/png/1607733/1663569331334-0b33878c-a49b-40d8-a82a-80d735139f6c.png#clientId=udf63137e-4aaa-4&errorMessage=unknown%20error&from=paste&id=u65cb3aba&originHeight=1010&originWidth=1080&originalType=url&ratio=1&rotation=0&showTitle=false&status=error&style=none&taskId=ucb736650-34fe-48d7-9b65-be4152cc10c&title=)Unlock
![](https://cdn.nlark.com/yuque/0/2022/png/1607733/1663569528947-2808e767-de2e-4fd0-8952-430121da6f48.png#clientId=udf63137e-4aaa-4&errorMessage=unknown%20error&from=paste&id=u9020c7fd&originHeight=906&originWidth=1080&originalType=url&ratio=1&rotation=0&showTitle=false&status=error&style=none&taskId=ub5b83823-548e-4556-80d4-8308af9306d&title=)
## Questions:

1. 如果Mutex已经被一个goroutine获取了锁，其他等待中的goroutine们只能一直等待，那么，等这个锁释放后，等待中的goroutine中哪一个会优先获取mutex？
2. Mutex的结构的演变过程？最终的state中每个字段代表什么意思？
3. 等待一个mutex的goroutine数最大是多少？能否满足现实需求？
4.  如果一个goroutine g1 通过Lock获取了锁， 在持有锁的期间， 另外一个goroutine g2 调用Unlock释放这个锁， 会出现什么现象？  
- g2 调用 Unlock panic
- g2 调用 Unlock 成功，但是如果将来 g1调用 Unlock 会 panic
- g2 调用 Unlock 成功，如果将来 g1调用 Unlock 也成功
## Answers:

1.  等待的goroutine们是以FIFO排队的
   -  1）当Mutex处于正常模式时，若此时没有新goroutine与队头goroutine竞争，则队头goroutine获得。若有新goroutine竞争大概率新goroutine获得。
   -  2）当队头goroutine竞争锁失败1ms后，它会将Mutex调整为饥饿模式。进入饥饿模式后，锁的所有权会直接从解锁goroutine移交给队头goroutine，此时新来的goroutine直接放入队尾。 
   - 3）当一个goroutine获取锁后，如果发现自己满足下列条件中的任何一个
      - 它是队列中最后一个
      - 它等待锁的时间少于1ms，则将锁切换回正常模式  
2. 如图所示：
```c
// 初版：
type Mutex struct {
        key  int32 // 锁是否被持有的标识
        sema int32 // 信号量专用，用以阻塞/唤醒goroutine
    }

// 给新人机会 && 多给些机会：
 type Mutex struct {
        state int32
        sema  uint32
    }

const (
        mutexLocked = 1 << iota // mutex is locked
        mutexWoken
        mutexWaiterShift = iota
    )

// 解决饥饿
type Mutex struct {
        state int32
        sema  uint32
    }
    
const (
        mutexLocked = 1 << iota // mutex is locked
        mutexWoken // 代表唤醒
        mutexStarving // 从state字段中分出一个饥饿标记
        mutexWaiterShift = iota // 代表位移
    
        starvationThresholdNs = 1e6
    )
```

3.  单从程序来看，可以支持 1<<(32-3) -1 ，约 0.5 Billion个， 其中32为state的类型int32，3位waiter字段的shift 考虑到实际goroutine初始化的空间为2K，0.5Billin*2K达到了1TB。
4. 

```go
package main
import (
    "sync"
    "time"
)
func main() {
    var mu sync.Mutex
    go func() {
        mu.Lock()
        time.Sleep(10 * time.Second)
        mu.Unlock()
    }()
    time.Sleep(time.Second)
    mu.Unlock()
    select {}
}
```
## RWMutex
### 定义

- A RWMutex is a reader/writer mutual exclusion lock.
- The lock can be held by any number of readers or a single writer, and
- a blocked writer also blocks new readers from acquiring the lock.
### 方法

1. Lock/Unlock：写操作时调用的方法。如果锁已经被 reader 或者 writer 持有，那么，Lock 方法会一直阻塞，直到能获取到锁；Unlock 则是配对的释放锁的方法。
2. RLock/RUnlock：读操作时调用的方法。如果锁已经被 writer 持有的话，RLock 方法会一直阻塞，直到能获取到锁，否则就直接返回；而 RUnlock 是 reader 释放锁的方法。
3. RLocker：这个方法的作用是为读操作返回一个 Locker 接口的对象。它的 Lock 方法会调用 RWMutex 的 RLock 方法，它的 Unlock 方法会调用 RWMutex 的 RUnlock 方法。

举例：使用 10 个 goroutine 进行读操作，每读取一次，sleep 1 毫秒，同时，还有一个 gorotine 进行写操作，每一秒写一次，这是一个 1 writer-n reader 的读写场景，而且写操作还不是很频繁（一秒一次）：
```go
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
RWMutex 包含一个 Mutex，以及四个辅助字段 writerSem、readerSem、readerCount 和 readerWait：
```go
type RWMutex struct {
  w           Mutex   // 互斥锁解决多个writer的竞争
  writerSem   uint32  // writer信号量
  readerSem   uint32  // reader信号量
  readerCount int32   // reader的数量
  readerWait  int32   // writer等待完成的reader的数量
}

const rwmutexMaxReaders = 1 << 30
```

- 字段 w：为 writer 的竞争锁而设计；
- 字段 readerCount：记录当前 reader 的数量（以及是否有 writer 竞争锁）；
- readerWait：记录 writer 请求锁时需要等待 read 完成的 reader 的数量；
- writerSem 和 readerSem：都是为了阻塞设计的信号量。
- rwmutexMaxReaders，定义了最大的 reader 数量。
### 实现
#### RLock/RUnLock
```go
func (rw *RWMutex) RLock() {
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
            // rw.readerCount是负值的时候，意味着此时有writer等待请求锁，因为writer优先级高，所以把后来的reader阻塞休眠
        runtime_SemacquireMutex(&rw.readerSem, false, 0)
    }
}
func (rw *RWMutex) RUnlock() {
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        rw.rUnlockSlow(r) // 有等待的writer
    }
}
func (rw *RWMutex) rUnlockSlow(r int32) {
    if atomic.AddInt32(&rw.readerWait, -1) == 0 {
        // 最后一个reader了，writer终于有机会获得锁了
        runtime_Semrelease(&rw.writerSem, false, 1)
    }
}
```

- 没有 writer 竞争或持有锁时，readerCount 和我们正常理解的 reader 的计数是一样的；
- 但是，如果有 writer 竞争锁或者持有锁时，那么，readerCount 不仅仅承担着 reader 的计数功能，还能够标识当前是否有 writer 竞争或持有锁，在这种情况下，请求锁的 reader 的处理进入第 4 行，阻塞等待锁的释放。

调用 RUnlock 的时候，我们需要将 Reader 的计数减去 1（第 8 行），因为 reader 的数量减少了一个。但是，第 8 行的 AddInt32 的返回值还有另外一个含义。如果它是负值，就表示当前有 writer 竞争锁，在这种情况下，还会调用 rUnlockSlow 方法，检查是不是 reader 都释放读锁了，如果读锁都释放了，那么可以唤醒请求写锁的 writer 了。
当一个或者多个 reader 持有锁的时候，竞争锁的 writer 会等待这些 reader 释放完，才可能持有这把锁。当 writer 请求锁的时候，是无法改变既有的 reader 持有锁的现实的，也不会强制这些 reader 释放锁，它的优先权只是限定后来的 reader 不要和它抢。所以，rUnlockSlow 将持有锁的 reader 计数减少 1 的时候，会检查既有的 reader 是不是都已经释放了锁，如果都释放了锁，就会唤醒 writer，让 writer 持有锁。
#### Lock
RWMutex 是一个多 writer 多 reader 的读写锁，所以同时可能有多个 writer 和 reader。那么，为了避免 writer 之间的竞争，RWMutex 就会使用一个 Mutex 来保证 writer 的互斥。
一旦一个 writer 获得了内部的互斥锁，就会反转 readerCount 字段，把它从原来的正整数 readerCount(>=0) 修改为负数（readerCount-rwmutexMaxReaders），让这个字段保持两个含义（既保存了 reader 的数量，又表示当前有 writer）。
我们来看下下面的代码。第 5 行，还会记录当前活跃的 reader 数量，所谓活跃的 reader，就是指持有读锁还没有释放的那些 reader。
```go
func (rw *RWMutex) Lock() {
    // 首先解决其他writer竞争问题
    rw.w.Lock()
    // 反转readerCount，告诉reader有writer竞争锁
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // 如果当前有reader持有锁，那么需要等待
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
}
```
如果 readerCount 不是 0，就说明当前有持有读锁的 reader，RWMutex 需要把这个当前 readerCount 赋值给 readerWait 字段保存下来（第 7 行）， 同时，这个 writer 进入阻塞等待状态（第 8 行）。
每当一个 reader 释放读锁的时候（调用 RUnlock 方法时），readerWait 字段就减 1，直到所有的活跃的 reader 都释放了读锁，才会唤醒这个 writer。
#### UnLock
当一个 writer 释放锁的时候，它会再次反转 readerCount 字段。可以肯定的是，因为当前锁由 writer 持有，所以，readerCount 字段是反转过的，并且减去了 rwmutexMaxReaders 这个常数，变成了负数。所以，这里的反转方法就是给它增加 rwmutexMaxReaders 这个常数值。
既然 writer 要释放锁了，那么就需要唤醒之后新来的 reader，不必再阻塞它们了，让它们开开心心地继续执行就好了。
在 RWMutex 的 Unlock 返回之前，需要把内部的互斥锁释放。释放完毕后，其他的 writer 才可以继续竞争这把锁。
```go
func (rw *RWMutex) Unlock() {
    // 告诉reader没有活跃的writer了
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    
    // 唤醒阻塞的reader们
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false, 0)
    }
    // 释放内部的互斥锁
    rw.w.Unlock()
}
```
## 互斥锁与读写锁的性能比较

- 读多写少
- 读少写多
- 读写一致
### 测试用例
#### Lock
```go
type RW interface {
	Write()
	Read()
}

const cost = time.Microsecond

type Lock struct {
	count int
	mu    sync.Mutex
}

func (l *Lock) Write() {
	l.mu.Lock()
	l.count++
	time.Sleep(cost)
	l.mu.Unlock()
}

func (l *Lock) Read() {
	l.mu.Lock()
	time.Sleep(cost)
	_ = l.count
	l.mu.Unlock()
}
```
#### RWLock
```go
type RWLock struct {
	count int
	mu    sync.RWMutex
}

func (l *RWLock) Write() {
	l.mu.Lock()
	l.count++
	time.Sleep(cost)
	l.mu.Unlock()
}

func (l *RWLock) Read() {
	l.mu.RLock()
	_ = l.count
	time.Sleep(cost)
	l.mu.RUnlock()
}
```
#### 基准测试
```go
func benchmark(b *testing.B, rw RW, read, write int) {
	for i := 0; i < b.N; i++ {
		var wg sync.WaitGroup
		for k := 0; k < read*100; k++ {
			wg.Add(1)
			go func() {
				rw.Read()
				wg.Done()
			}()
		}
		for k := 0; k < write*100; k++ {
			wg.Add(1)
			go func() {
				rw.Write()
				wg.Done()
			}()
		}
		wg.Wait()
	}
}


func BenchmarkReadMore(b *testing.B)    { benchmark(b, &Lock{}, 9, 1) }
func BenchmarkReadMoreRW(b *testing.B)  { benchmark(b, &RWLock{}, 9, 1) }
func BenchmarkWriteMore(b *testing.B)   { benchmark(b, &Lock{}, 1, 9) }
func BenchmarkWriteMoreRW(b *testing.B) { benchmark(b, &RWLock{}, 1, 9) }
func BenchmarkEqual(b *testing.B)       { benchmark(b, &Lock{}, 5, 5) }
func BenchmarkEqualRW(b *testing.B)     { benchmark(b, &RWLock{}, 5, 5) }
```

- 三种场景，分别使用 Lock 和 RWLock 测试，共 6 个用例。
- 每次测试读写操作合计 1000 次，例如读多写少场景，读 900 次，写 100 次。
- 使用 sync.WaitGroup 阻塞直到读写操作全部运行结束。



