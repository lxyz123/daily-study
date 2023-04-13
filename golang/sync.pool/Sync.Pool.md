## 背景
Go 是一个自动垃圾回收的编程语言，采用[三色并发标记算法](https://go.dev/blog/ismmkeynote)标记对象并回收。和其它没有自动垃圾回收的编程语言不同，使用 Go 语言创建对象的时候，我们没有回收 / 释放的心理负担，想用就用，想创建就创建。
但是，如果想使用 Go 开发一个高性能的应用程序的话，就必须考虑垃圾回收给性能带来的影响，毕竟，Go 的自动垃圾回收机制还是有一个 STW（stop-the-world，程序暂停）的时间，而且，大量地创建在堆上的对象，也会影响垃圾回收标记的时间。
所以，一般做性能优化的时候，会采用对象池的方式，把不用的对象回收起来，避免被垃圾回收掉，这样使用的时候就不必在堆上重新创建了。
不止如此，像数据库连接、TCP 的长连接，这些连接在创建的时候是一个非常耗时的操作。如果每次都创建一个新的连接对象，耗时较长，很可能整个业务的大部分耗时都花在了创建连接上。
所以，如果能把这些连接保存下来，避免每次使用的时候都重新创建，不仅可以大大减少业务的耗时，还能提高应用程序的整体性能。
**它池化的对象可能会被垃圾回收掉，这对于数据库长连接等场景是不合适的。**
## 定义
sync.Pool 数据类型用来保存一组可独立访问的临时对象。它池化的对象会在未来的某个时候被毫无预兆地移除掉。而且，如果没有别的对象引用这个被移除的对象的话，这个被移除的对象就会被垃圾回收掉。
 对于很多需要重复分配、回收内存的地方，sync.Pool 是一个很好的选择。频繁地分配、回收内存会给 GC 带来一定的负担，严重的时候会引起 CPU 的毛刺，而 sync.Pool 可以将暂时不用的对象缓存起来，待下次需要的时候直接使用，不用再次经过内存分配，复用对象的内存，减轻 GC 的压力，提升系统的性能。  
## 适用场景

-  当多个 goroutine 都需要创建同⼀个对象的时候，如果 goroutine 数过多，导致对象的创建数⽬剧增，进⽽导致 GC 压⼒增大。形成 “并发⼤－占⽤内存⼤－GC 缓慢－处理并发能⼒降低－并发更⼤”这样的恶性循环。  
-  在这个时候，需要有⼀个对象池，每个 goroutine 不再⾃⼰单独创建对象，⽽是从对象池中获取出⼀个对象（如果池中已经有的话）  。
## 举例
```go
package main
import (
    "fmt"
    "sync"
)

var pool *sync.Pool

type Person struct {
    Name string
}

func initPool() {
    pool = &sync.Pool {
        New: func()interface{} {
            fmt.Println("Creating a new Person")
            return new(Person)
        },
    }
}

func main() {
    initPool()

    p := pool.Get().(*Person)
    fmt.Println("首次从 pool 里获取：", p)

    p.Name = "first"
    fmt.Printf("设置 p.Name = %s\n", p.Name)

    pool.Put(p)

    fmt.Println("Pool 里已有一个对象：&{first}，调用 Get: ", pool.Get().(*Person))
    fmt.Println("Pool 没有对象了，调用 Get: ", pool.Get().(*Person))
}

```
运行结果：
```go
Creating a new Person
首次从 pool 里获取： &{}
设置 p.Name = first
Pool 里已有一个对象：&{first}，Get:  &{first}
Creating a new Person
Pool 没有对象了，Get:  &{}
```
首先，需要初始化 Pool，唯一需要的就是设置好 New 函数。当调用 Get 方法时，如果池子里缓存了对象，就直接返回缓存的对象。如果没有存货，则调用 New 函数创建一个新的对象。
Get 方法取出来的对象和上次 Put 进去的对象实际上是同一个，Pool 没有做任何“清空”的处理。但我们不应当对此有任何假设，因为在实际的并发使用场景中，无法保证这种顺序，最好的做法是在 Put 前，将对象清空。
## 具体使用
### fmt
```go
func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}

func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
	p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}
```
Fprintf 函数的参数是一个 io.Writer，Printf 传的是 os.Stdout，相当于直接输出到标准输出。这里的 newPrinter 用的就是 Pool：  
```go
// newPrinter allocates a new pp struct or grabs a cached one.
func newPrinter() *pp {
	p := ppFree.Get().(*pp)
	p.panicking = false
	p.erroring = false
	p.wrapErrs = false
	p.fmt.init(&p.buf)
	return p
}

var ppFree = sync.Pool{
	New: func() interface{} { return new(pp) },
}

```
 回到 Fprintf 函数，拿到 pp 指针后，会做一些 format 的操作，并且将 p.buf 里面的内容写入 w。最后，调用 free 函数，将 pp 指针归还到 Pool 中：  
```go
// free saves used pp structs in ppFree; avoids an allocation per invocation.
func (p *pp) free() {
	if cap(p.buf) > 64<<10 {
		return
	}

	p.buf = p.buf[:0]
	p.arg = nil
	p.value = reflect.Value{}
	p.wrappedErr = nil
	ppFree.Put(p)
}

```
### Gin
 	对 context 取用也到了 sync.Pool
```go
engine.pool.New = func() interface{} {
	return engine.allocateContext()
}

func (engine *Engine) allocateContext() *Context {
	return &Context{engine: engine, KeysMutex: &sync.RWMutex{}}
}

// ServeHTTP conforms to the http.Handler interface.
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)

	engine.pool.Put(c)
}
```
 先调用 Get 取出来缓存的对象，然后会做一些 reset 操作，再执行 handleHTTPRequest，最后再 Put 回 Pool。  
## 使用方法
### New
Pool struct 包含一个 New 字段，这个字段的类型是函数 func() interface{}。当调用 Pool 的 Get 方法从池中获取元素，没有更多的空闲元素可返回时，就会调用这个 New 方法来创建新的元素。如果你没有设置 New 字段，没有更多的空闲元素可返回时，Get 方法将返回 nil，表明当前没有可用的元素。有趣的是，New 是可变的字段。这就意味着，你可以在程序运行的时候改变创建元素的方法。当然，很少有人会这么做，因为一般我们创建元素的逻辑都是一致的，要创建的也是同一类的元素，所以你在使用 Pool 的时候也没必要玩一些“花活”，在程序运行时更改 New 的值。
### Get
如果调用这个方法，就会从 Pool取走一个元素，这也就意味着，这个元素会从 Pool 中移除，返回给调用者。不过，除了返回值是正常实例化的元素，Get 方法的返回值还可能会是一个 nil（Pool.New 字段没有设置，又没有空闲元素可以返回），所以你在使用的时候，可能需要判断。
### Put
这个方法用于将一个元素返还给 Pool，Pool 会把这个元素保存到池中，并且可以复用。但如果 Put 一个 nil 值，Pool 就会忽略这个值。好了，了解了这几个方法，下面我们看看 sync.Pool 最常用的一个场景：buffer 池（缓冲池）。因为 byte slice 是经常被创建销毁的一类对象，使用 buffer 池可以缓存已经创建的 byte slice，比如，著名的静态网站生成工具 Hugo 中，就包含这样的实现[bufpool](https://github.com/gohugoio/hugo/blob/master/bufferpool/bufpool.go)。
```go

var buffers = sync.Pool{
  New: func() interface{} { 
    return new(bytes.Buffer)
  },
}

func GetBuffer() *bytes.Buffer {
  return buffers.Get().(*bytes.Buffer)
}

func PutBuffer(buf *bytes.Buffer) {
  buf.Reset()
  buffers.Put(buf)
}
```
## 实现原理
Go 1.13 之前的 sync.Pool 的实现有 2 大问题：

1. **每次 GC 都会回收创建的对象。**

如果缓存元素数量太多，就会导致 STW 耗时变长；缓存元素都被回收后，会导致 Get 命中率下降，Get 方法不得不新创建很多对象。

2. **底层实现使用了 Mutex，对这个锁并发请求竞争激烈的时候，会导致性能的下降。**

在 Go 1.13 中，sync.Pool 做了大量的优化。前几讲中我提到过，提高并发程序性能的优化点是尽量不要使用锁，如果不得已使用了锁，就把锁 Go 的粒度降到最低。Go 对 Pool 的优化就是避免使用锁，同时将加锁的 queue 改成 lock-free 的 queue 的实现，给即将移除的元素再多一次“复活”的机会。
sync.Pool数据结构如下：
![f4003704663ea081230760098f8af696.webp](https://cdn.nlark.com/yuque/0/2022/webp/1607733/1666661307932-775f2773-d01e-4000-85ec-44c4d965e93d.webp#clientId=u26bfc6b5-d5a3-4&from=paste&height=2186&id=uc47e49a2&name=f4003704663ea081230760098f8af696.webp&originHeight=2186&originWidth=3659&originalType=binary&ratio=1&rotation=0&showTitle=false&size=90794&status=done&style=none&taskId=u3144fdb6-b6d1-440a-a4bc-bb86d952003&title=&width=3659)
Pool 最重要的两个字段是 local 和 victim，它们两个主要用来存储空闲的元素。每次垃圾回收的时候，Pool 会把 victim 中的对象移除，然后把 local 的数据给 victim，这样的话，local 就会被清空，而 victim 就像一个垃圾分拣站，里面的东西可能会被当做垃圾丢弃了，但是里面有用的东西也可能被捡回来重新使用。
victim 中的元素如果被 Get 取走，那么这个元素就很幸运，因为它又“活”过来了。但是，如果这个时候 Get 的并发不是很大，元素没有被 Get 取走，那么就会被移除掉，因为没有别人引用它的话，就会被垃圾回收掉。
垃圾回收时sync.Pool的处理逻辑：
```go

func poolCleanup() {
    // 丢弃当前victim, STW所以不用加锁
    for _, p := range oldPools {
        p.victim = nil
        p.victimSize = 0
    }

    // 将local复制给victim, 并将原local置为nil
    for _, p := range allPools {
        p.victim = p.local
        p.victimSize = p.localSize
        p.local = nil
        p.localSize = 0
    }

    oldPools, allPools = allPools, nil
}
```
所有当前主要的空闲可用的元素都存放在 local 字段中，请求元素时也是优先从 local 字段中查找可用的元素。local 字段包含一个 poolLocalInternal 字段，并提供 CPU 缓存对齐，从而避免 false sharing。而 poolLocalInternal 也包含两个字段：private 和 shared。

- private，代表一个缓存的元素，而且只能由相应的一个 P 存取。因为一个 P 同时只能执行一个 goroutine，所以不会有并发的问题。
- shared，可以由任意的 P 访问，但是只有本地的 P 才能 pushHead/popHead，其它 P 可以 popTail，相当于只有一个本地的 P 作为生产者（Producer），多个 P 作为消费者（Consumer），它是使用一个 local-free 的 queue 列表实现的。
### Get
```go

func (p *Pool) Get() interface{} {
    // 把当前goroutine固定在当前的P上
    l, pid := p.pin()
    x := l.private // 优先从local的private字段取，快速
    l.private = nil
    if x == nil {
        // 从当前的local.shared弹出一个，注意是从head读取并移除
        x, _ = l.shared.popHead()
        if x == nil { // 如果没有，则去偷一个
            x = p.getSlow(pid) 
        }
    }
    runtime_procUnpin()
    // 如果没有获取到，尝试使用New函数生成一个新的
    if x == nil && p.New != nil {
        x = p.New()
    }
    return x
}
```

1. 首先，调用 p.pin() 函数将当前的 goroutine 和 P 绑定，禁止被抢占，返回当前 P 对应的 poolLocal，以及 pid。
2. 然后直接取 l.private，赋值给 x，并置 l.private 为 nil。
3. 判断 x 是否为空，若为空，则尝试从 l.shared 的头部 pop 一个对象出来，同时赋值给 x。
4. 如果 x 仍然为空，则调用 getSlow 尝试从其他 P 的 shared 双端队列尾部“偷”一个对象出来。
5. Pool 的相关操作做完了，调用 runtime_procUnpin() 解除非抢占。
6. 最后如果还是没有取到缓存的对象，那就直接调用预先设置好的 New 函数，创建一个出来。

![20200418091935.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1666663216625-67f501eb-b372-4735-bf1c-8f982aae5675.png#clientId=u26bfc6b5-d5a3-4&from=paste&height=1622&id=uf33ffe26&name=20200418091935.png&originHeight=1622&originWidth=2080&originalType=binary&ratio=1&rotation=0&showTitle=false&size=255755&status=done&style=none&taskId=u545c750d-57c6-470f-8aaf-a007593985f&title=&width=2080)
#### pin
```go
// src/sync/pool.go

// 调用方必须在完成取值后调用 runtime_procUnpin() 来取消抢占。
func (p *Pool) pin() (*poolLocal, int) {
	pid := runtime_procPin()
	s := atomic.LoadUintptr(&p.localSize) // load-acquire
	l := p.local                          // load-consume
	// 因为可能存在动态的 P（运行时调整 P 的个数）
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	return p.pinSlow()
}

```
这里的重点是 getSlow 方法，我们来分析下。看名字也就知道了，它的耗时可能比较长。它首先要遍历所有的 local，尝试从它们的 shared 弹出一个元素。如果还没找到一个，那么，就开始对 victim 下手了。在 vintim 中查询可用元素的逻辑还是一样的，先从对应的 victim 的 private 查找，如果查不到，就再从其它 victim 的 shared 中查找。
 getSlow 方法的主要逻辑：
```go

func (p *Pool) getSlow(pid int) interface{} {

    size := atomic.LoadUintptr(&p.localSize)
    locals := p.local                       
    // 从其它proc中尝试偷取一个元素
    for i := 0; i < int(size); i++ {
        l := indexLocal(locals, (pid+i+1)%int(size))
        if x, _ := l.shared.popTail(); x != nil {
            return x
        }
    }

    // 如果其它proc也没有可用元素，那么尝试从vintim中获取
    size = atomic.LoadUintptr(&p.victimSize)
    if uintptr(pid) >= size {
        return nil
    }
    locals = p.victim
    l := indexLocal(locals, pid)
    if x := l.private; x != nil { // 同样的逻辑，先从vintim中的local private获取
        l.private = nil
        return x
    }
    for i := 0; i < int(size); i++ { // 从vintim其它proc尝试偷取
        l := indexLocal(locals, (pid+i)%int(size))
        if x, _ := l.shared.popTail(); x != nil {
            return x
        }
    }

    // 如果victim中都没有，则把这个victim标记为空，以后的查找可以快速跳过了
    atomic.StoreUintptr(&p.victimSize, 0)

    return nil
}
```
pin 方法会将此 goroutine 固定在当前的 P 上，避免查找元素期间被其它的 P 执行。固定的好处就是查找元素期间直接得到跟这个 P 相关的 local。有一点需要注意的是，pin 方法在执行的时候，如果跟这个 P 相关的 local 还没有创建，或者运行时 P 的数量被修改了的话，就会新创建 local。
### Put
```go

func (p *Pool) Put(x interface{}) {
    if x == nil { // nil值直接丢弃
        return
    }
    l, _ := p.pin()
    if l.private == nil { // 如果本地private没有值，直接设置这个值即可
        l.private = x
        x = nil
    }
    if x != nil { // 否则加入到本地队列中
        l.shared.pushHead(x)
    }
    runtime_procUnpin()
}
```
Put 的逻辑相对简单，优先设置本地 private，如果 private 字段已经有值了，那么就把此元素 push 到本地队列中。
## 坑


