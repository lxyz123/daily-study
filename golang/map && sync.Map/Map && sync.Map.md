# Map
# 设计原理
## 哈希函数（O(n)，O(1)）
![image.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1665990753547-a87d1a60-b2f2-4b77-b09f-755f0faf468a.png#clientId=ubae1b5de-71bd-4&errorMessage=unknown%20error&from=paste&id=u37d62bd1&name=image.png&originHeight=460&originWidth=1200&originalType=url&ratio=1&rotation=0&showTitle=false&size=65895&status=error&style=none&taskId=u96e39d0f-7fd7-449d-9dd9-a7b9554eb9d&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1665990768912-b1c0b12a-40bb-4fe4-a8f6-366de587df57.png#clientId=ubae1b5de-71bd-4&errorMessage=unknown%20error&from=paste&id=ua1ca0807&name=image.png&originHeight=460&originWidth=1200&originalType=url&ratio=1&rotation=0&showTitle=false&size=52163&status=error&style=none&taskId=u359526b9-260f-49fe-8a56-ce9aca2ddb3&title=)
## 冲突解决
哈希函数输入的范围一定会远远大于输出的范围，所以在使用哈希表时一定会遇到冲突，哪怕我们使用了完美的哈希函数，当输入的键足够多也会产生冲突。然而多数的哈希函数都是不够完美的，所以仍然存在发生哈希碰撞的可能，这时就需要一些方法来解决哈希碰撞的问题，常见方法的就是开放寻址法和拉链法。
### 开放寻址法
[开放寻址法](https://en.wikipedia.org/wiki/Open_addressing)是一种在哈希表中解决哈希碰撞的方法，这种方法的核心思想是**依次探测和比较数组中的元素以判断目标键值对是否存在于哈希表中**，如果我们使用开放寻址法来实现哈希表，那么实现哈希表底层的数据结构就是数组，当我们向当前哈希表写入新的数据时，如果发生了冲突，就会将键值对写入到下一个索引不为空的位置
### 拉链法

1. 找到键相同的键值对 — 更新键对应的值；
2. 没有找到键相同的键值对 — 在链表的末尾追加新的键值对

![image.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1665991968856-a6c605de-ba30-4136-bce9-2ab25e8ffd7a.png#clientId=ubae1b5de-71bd-4&errorMessage=unknown%20error&from=paste&id=u5ca2c256&name=image.png&originHeight=630&originWidth=1200&originalType=url&ratio=1&rotation=0&showTitle=false&size=64461&status=error&style=none&taskId=ud60c25fd-7672-4fd2-a585-5e382fce7f0&title=)
# 数据结构
```go
// A header for a Go map.
type hmap struct {
	count     int // 元素个数，调用 len(map) 时，直接返回此值
	flags     uint8
    // B 表示当前哈希表持有的 buckets 数量，但是因为哈希表中桶的数量都是 2 的倍数，所以该字段会存储对数，也就是 len(buckets) == 2^B
	B         uint8 
	// overflow 的 bucket 近似数
	noverflow uint16
    // hash0 是哈希的种子，它能为哈希函数的结果引入随机性，这个值在创建哈希表时确定，并在调用哈希函数时作为参数传入
	hash0     uint32
    // 指向 buckets 数组，大小为 2^B
    // 如果元素个数为0，就为 nil
	buckets    unsafe.Pointer
	// 扩容的时候，buckets 长度会是 oldbuckets 的两倍
	oldbuckets unsafe.Pointer
	// 指示扩容进度，小于此地址的 buckets 迁移完成
	nevacuate  uintptr
	extra *mapextra // optional fields
}

type bmap struct {
	tophash [bucketCnt]uint8
}

type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}

```

![57576986-acd87600-749f-11e9-8710-75e423c7efdb.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1665992433895-a579ed45-9458-4162-b265-89573256a312.png#clientId=ubae1b5de-71bd-4&errorMessage=unknown%20error&from=paste&height=1558&id=u9132a96e&name=57576986-acd87600-749f-11e9-8710-75e423c7efdb.png&originHeight=1558&originWidth=2248&originalType=binary&ratio=1&rotation=0&showTitle=false&size=242913&status=error&style=none&taskId=ue61977b8-0811-41d5-817e-a71d6ded082&title=&width=2248)
```go
          ┌─────────────┐                                                                                                                                                                                                           
          │    hmap     │                                                                                                                                                                                                           
          ├─────────────┴──────────────────┐           ┌───────────────┐                                        ┌─────────┐                               ┌─────────┐                                                               
          │           count int            │           │               │                     ┌─────────────────▶│  bmap   │                          ┌───▶│  bmap   │                                                               
          │                                │           │               ▼                     │                  ├─────────┴─────────────────────┐    │    ├─────────┴─────────────────────┐                                         
          ├────────────────────────────────┤           │    ────────┬─────┐                  │                  │   tophash [bucketCnt]uint8    │    │    │   tophash [bucketCnt]uint8    │                                         
          │          flags uint8           │           │       ▲    │  0  │                  │                  │                               │    │    │                               │                                         
          │                                │           │       │    │     │──────────────────┘                  ├──────────┬────────────────────┤    │    ├──────────┬────────────────────┤                                         
          ├────────────────────────────────┤           │       │    ├─────┤                                     │   keys   │                    │    │    │   keys   │                    │                                         
          │            B uint8             │           │       │    │  1  │                                     ├───┬───┬──┴┬───┬───┬───┬───┬───┤    │    ├───┬───┬──┴┬───┬───┬───┬───┬───┤                                         
          │                                │           │       │    │     │──────────────────┐                  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │    │    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                         
          ├────────────────────────────────┤           │       │    ├─────┤                  │                  ├───┴───┴──┬┴───┴───┴───┴───┴───┤    │    ├───┴───┴──┬┴───┴───┴───┴───┴───┤                                         
          │        noverflow uint16        │           │       │    │  2  │                  │                  │  values  │                    │    │    │  values  │                    │                                         
          │                                │           │       │    │     │                  │                  ├───┬───┬──┴┬───┬───┬───┬───┬───┤    │    ├───┬───┬──┴┬───┬───┬───┬───┬───┤                                         
          ├────────────────────────────────┤           │       │    ├─────┤                  │                  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │    │    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                         
          │          hash0 uint32          │           │       │    │  3  │                  │                  ├───┴───┴───┴───┴───┴───┴───┴───┤    │    ├───┴───┴───┴───┴───┴───┴───┴───┤                                         
          │                                │           │       │    │     │                  │                  │        overflow *bmap         │    │    │        overflow *bmap         │                                         
          ├────────────────────────────────┤           │       │    ├─────┤                  │                  │                               │────┘    │                               │                                         
          │     buckets unsafe.Pointer     │           │       │    │  4  │                  │                  ├─────────┬─────────────────────┘         └───────────────────────────────┘                                         
          │                                │───────────┘       │    │     │                  └─────────────────▶│  bmap   │                                                                                                         
          ├────────────────────────────────┤                        ├─────┤                                     ├─────────┴─────────────────────┐                                                                                   
          │   oldbuckets unsafe.Pointer    │                        │  5  │                                     │   tophash [bucketCnt]uint8    │                                                                                   
          │                                │                        │     │                                     │                               │                                                                                   
          ├────────────────────────────────┤         size = 2 ^ B   ├─────┤                                     ├──────────┬────────────────────┤                                                                                   
          │       nevacuate uintptr        │                        │  6  │                                     │   keys   │                    │                                                                                   
          │                                │                        │     │                                     ├───┬───┬──┴┬───┬───┬───┬───┬───┤                                                                                   
          ├────────────────────────────────┤                   │    ├─────┤                                     │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                                                                   
          │        extra *mapextra         │                   │    │  7  │                                     ├───┴───┴──┬┴───┴───┴───┴───┴───┤                                                                                   
       ┌──│                                │                   │    │     │                                     │  values  │                    │                                                                                   
       │  └────────────────────────────────┘                   │    └─────┘                                     ├───┬───┬──┴┬───┬───┬───┬───┬───┤                                                                                   
       │                                                       │      ....                                      │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                                                                   
       │                                                       │                                                ├───┴───┴───┴───┴───┴───┴───┴───┤                                                                                   
       │                                                       │    ┌─────┐                                     │        overflow *bmap         │                                                                                   
       │                                                       │    │ 61  │                                     │                               │                                                                                   
       │                                                       │    │     │                                     └───────────────────────────────┘                                                                                   
       ▼                                                       │    ├─────┤                                               ............                                                                                              
┌─────────────┐                                                │    │ 62  │                                     ┌─────────┐                               ┌─────────┐                              ┌─────────┐                      
│  mapextra   │                                                │    │     │────────────────────────────────────▶│  bmap   │                          ┌───▶│  bmap   │                         ┌───▶│  bmap   │                      
├─────────────┴──────────────┐                                 │    ├─────┤                                     ├─────────┴─────────────────────┐    │    ├─────────┴─────────────────────┐   │    ├─────────┴─────────────────────┐
│     overflow *[]*bmap      │                                 │    │ 63  │                                     │   tophash [bucketCnt]uint8    │    │    │   tophash [bucketCnt]uint8    │   │    │   tophash [bucketCnt]uint8    │
│                            │                                 ▼    │     │──────────────────┐                  │                               │    │    │                               │   │    │                               │
├────────────────────────────┤                              ────────┴─────┘                  │                  ├──────────┬────────────────────┤    │    ├──────────┬────────────────────┤   │    ├──────────┬────────────────────┤
│    oldoverflow *[]*bmap    │                                                               │                  │   keys   │                    │    │    │   keys   │                    │   │    │   keys   │                    │
│                            │                                                               │                  ├───┬───┬──┴┬───┬───┬───┬───┬───┤    │    ├───┬───┬──┴┬───┬───┬───┬───┬───┤   │    ├───┬───┬──┴┬───┬───┬───┬───┬───┤
├────────────────────────────┤                                                               │                  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │    │    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │   │    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │
│     nextoverflow *bmap     │                                                               │                  ├───┴───┴──┬┴───┴───┴───┴───┴───┤    │    ├───┴───┴──┬┴───┴───┴───┴───┴───┤   │    ├───┴───┴──┬┴───┴───┴───┴───┴───┤
│                            │                                                               │                  │  values  │                    │    │    │  values  │                    │   │    │  values  │                    │
└────────────────────────────┘                                                               │                  ├───┬───┬──┴┬───┬───┬───┬───┬───┤    │    ├───┬───┬──┴┬───┬───┬───┬───┬───┤   │    ├───┬───┬──┴┬───┬───┬───┬───┬───┤
                                                                                             │                  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │    │    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │   │    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │
                                                                                             │                  ├───┴───┴───┴───┴───┴───┴───┴───┤    │    ├───┴───┴───┴───┴───┴───┴───┴───┤   │    ├───┴───┴───┴───┴───┴───┴───┴───┤
                                                                                             │                  │        overflow *bmap         │    │    │        overflow *bmap         │   │    │        overflow *bmap         │
                                                                                             │                  │                               │────┘    │                               │───┘    │                               │
                                                                                             │                  ├─────────┬─────────────────────┘         └───────────────────────────────┘        └───────────────────────────────┘
                                                                                             └─────────────────▶│  bmap   │                                                                                                         
                                                                                                                ├─────────┴─────────────────────┐                                                                                   
                                                                                                                │   tophash [bucketCnt]uint8    │                                                                                   
                                                                                                                │                               │                                                                                   
                                                                                                                ├──────────┬────────────────────┤                                                                                   
                                                                                                                │   keys   │                    │                                                                                   
                                                                                                                ├───┬───┬──┴┬───┬───┬───┬───┬───┤                                                                                   
                                                                                                                │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                                                                   
                                                                                                                ├───┴───┴──┬┴───┴───┴───┴───┴───┤                                                                                   
                                                                                                                │  values  │                    │                                                                                   
                                                                                                                ├───┬───┬──┴┬───┬───┬───┬───┬───┤                                                                                   
                                                                                                                │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                                                                   
                                                                                                                ├───┴───┴───┴───┴───┴───┴───┴───┤                                                                                   
                                                                                                                │        overflow *bmap         │                                                                                   
                                                                                                                │                               │                                                                                   
                                                                                                                └───────────────────────────────┘                                                                                   
```
![57577391-f88f1d80-74a7-11e9-893c-4783dc4fb35e.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1666161452324-27ed67f2-7b12-4a18-b9e7-1f6a97db9802.png#clientId=uc8fbaacb-194e-4&errorMessage=unknown%20error&from=paste&height=1668&id=u2d5eb561&name=57577391-f88f1d80-74a7-11e9-893c-4783dc4fb35e.png&originHeight=1668&originWidth=1390&originalType=binary&ratio=1&rotation=0&showTitle=false&size=135713&status=error&style=none&taskId=uae24d935-b6ec-48dd-a0ac-adb189203a4&title=&width=1390)
```go
map[int64]int8
```
按照 key/value/key/value/... 这样的模式存储，那在每一个 key/value 对之后都要额外 padding 7 个字节；而将所有的 key，value 分别绑定到一起，这种形式 key/key/.../value/value/...，则只需要在最后添加 padding。
每个 bucket 设计成最多只能放 8 个 key-value 对，如果有第 9 个 key-value 落入当前的 bucket，那就需要再构建一个 bucket ，通过 overflow 指针连接起来。
## 初始化
### 创建map
```go
ageMp := make(map[string]int)
// 指定 map 长度
ageMp := make(map[string]int, 8)

// ageMp 为 nil，不能向其添加元素，会直接panic
var ageMp map[string]int

```
make(map[k]v, hint，在 hint <= 8(bucksize)时，会调用 makemap_small 来进行初始化，如果 hint > 8， 则调用makemap
```go
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}

```

1. 计算哈希占用的内存是否溢出或者超出能分配的最大值；
2. 调用 [runtime.fastrand](https://draveness.me/golang/tree/runtime.fastrand) 获取一个随机的哈希种子；
3. 根据传入的 hint 计算出需要的最小需要的桶的数量；
4. 使用 [runtime.makeBucketArray](https://draveness.me/golang/tree/runtime.makeBucketArray) 创建用于保存桶的数组；

根据传入的 B 计算出的需要创建的桶数量并在内存中分配一片连续的空间用于存储数据：
```go
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	if b >= 4 {
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	buckets = newarray(t.bucket, int(nbuckets))
	if base != nbuckets {
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

- 当桶的数量小于  时，由于数据较少、使用溢出桶的可能性较低，会省略创建的过程以减少额外开销；
- 当桶的数量多于  时，会额外创建 个溢出桶；

根据上述代码，我们能确定在正常情况下，正常桶和溢出桶在内存中的存储空间是连续的，只是被 [runtime.hmap](https://draveness.me/golang/tree/runtime.hmap) 中的不同字段引用，当溢出桶数量较多时会通过 [runtime.newobject](https://draveness.me/golang/tree/runtime.newobject) 创建新的溢出桶。
### key定位过程
key 经过哈希计算后得到哈希值，共 64 个 bit 位（64位机，32位机就不讨论了，现在主流都是64位机），计算它到底要落在哪个桶时，只会用到最后 B 个 bit 位。还记得前面提到过的 B 吗？如果 B = 5，那么桶的数量，也就是 buckets 数组的长度是 2^5 = 32。
例如，现在有一个 key 经过哈希函数计算后，得到的哈希结果是：
```shell
 10010111 | 000011110110110010001111001010100010010110010101010 │ 01010
```
用最后的 5 个 bit 位，也就是 01010，值为 10，也就是 10 号桶。这个操作实际上就是取余操作，但是取余开销太大，所以代码实现上用的位操作代替。
再用哈希值的高 8 位，找到此 key 在 bucket 中的位置，这是在寻找已有的 key。最开始桶内还没有 key，新加入的 key 会找到第一个空位，放入。
buckets 编号就是桶编号，当两个不同的 key 落在同一个桶中，也就是发生了哈希冲突。冲突的解决手段是用链表法：在 bucket 中，从前往后找到第一个空位。这样，在查找某个 key 时，先找到对应的桶，再去遍历 bucket 中的 key。![57577721-faf57580-74af-11e9-8826-aacdb34a1d2b.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1666171104232-723579d7-0f50-4cb5-91bd-59b21875a77f.png#clientId=ucdd43824-96e5-4&errorMessage=unknown%20error&from=paste&height=2248&id=u9654479c&name=57577721-faf57580-74af-11e9-8826-aacdb34a1d2b.png&originHeight=2248&originWidth=1766&originalType=binary&ratio=1&rotation=0&showTitle=false&size=181006&status=error&style=none&taskId=u88c1c51b-6134-408a-953e-8dcc1ec25fe&title=&width=1766)
假定 B = 5，所以 bucket 总数就是 2^5 = 32。首先计算出待查找 key 的哈希，使用低 5 位 00110，找到对应的 6 号 bucket，使用高 8 位 10010111，对应十进制 151，在 6 号 bucket 中寻找 tophash 值（HOB hash）为 151 的 key，找到了 2 号槽位，这样整个查找过程就结束了。
如果在 bucket 中没找到，并且 overflow 不为空，还要继续去 overflow bucket 中寻找，直到找到或是所有的 key 槽位都找遍了，包括所有的 overflow bucket。
```shell
   ┌─────────────┬─────────────────────────────────────────────────────┬─────────────┐                          
   │  10010111   │ 000011110110110010001111001010100010010110010101010 │    01010    │                          
   └─────────────┴─────────────────────────────────────────────────────┴─────────────┘                          
          ▲                                                                   ▲                                 
          │                                                                   │                                 
          │                                                                   │                   B = 5         
          │                                                                   │             bucketMask = 11111  
          │                                                                   │                                 
          │                                                            ┌─────────────┬───┬─────────────┐        
          │                                                            │  low bits   │ & │ bucketMask  │        
          │                                                            ├─────────────┴───┴─────────────┤        
          │                                                            │         01010 & 11111         │        
          │                                                            └───────────────────────────────┘        
          │                                                                            │                        
          │                                                                            │                        
          │                                                                            │                        
          │                                                                            ▼                        
          │                                                                  ┌───────────────────┐              
          │                                                                  │    01010 = 12     │              
          │                                                                  └───────────────────┘              
          │                                                                            │                        
          │                                                                            │                        
          │                                                                            │                        
          │                                                                            │                        
          │                                                                            │                        
   ┌─────────────┐                                                                     │                        
   │   tophash   │                                                                     │                        
   └─────────────┘                       ┌─────────┐                                   │                        
          │                              │ buckets │                                   ▼                        
          │                              ├───┬───┬─┴─┬───┬───┬───┬───┬───┬───┬───┬───┬───┐    ┌───┬───┬───┐     
          │                              │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │ ...│29 │30 │31 │     
          │                              └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘    └───┴───┴───┘     
          ▼                                                                            │                        
┌──────────────────┐                                                                   │                        
│  10010111 = 151  │─────────────┐                                                     │                        
└──────────────────┘             │                                                     │                        
                                 │                                                     │                        
                                 │                                                     │                        
                                 │                             ┌───────────────────────┘                        
                                 │                             │                                                
                                 │                             │                                                
                                 │                             │                                                
                                 │                             │                                                
                                 │                             │                                                
                                 │                             │                                                
                                 │                             ▼                                                
                                 │                      ┌─────────────┐                                         
                                 │                      │   bucket    │                                         
   ┌─────────────────────────────┼──────────────────────┴─────────────┤                                         
   │                             │                                    │                                         
   │ ┌─────────┐                 │                                    │                                         
   │ │ tophash │                 ▼                                    │                                         
   │ ├───────┬─┴─────┬───────┬───────┬───────┬───────┬───────┬───────┐│                                         
   │ │  124  │  33   │  41   │  151  │   1   │   0   │   0   │   0   ││                                         
   │ ├───────┴─┬─────┴───────┴───────┴───────┴───────┴───────┴───────┘│                                         
   │ │    1    │                                                      │                                         
   │ ├───────┬─┴─────┬───────┬───────┬───────┬───────┬───────┬───────┐│                                         
   │ │  124  │  412  │  423  │   5   │  14   │   0   │   0   │   0   ││                                         
   │ ├───────┴─┬─────┴───────┴───────┴───────┴───────┴───────┴───────┘│                                         
   │ │ values  │                                                      │                                         
   │ ├───────┬─┴─────┬───────┬───────┬───────┬───────┬───────┬───────┐│                                         
   │ │  v0   │  v1   │  v2   │  v3   │  v4   │   0   │   0   │   0   ││                                         
   │ ├───────┴───────┴──┬────┴───────┴───────┴───────┴───────┴───────┘│                                         
   │ │ overflow pointer │                                             │                                         
   │ └──────────────────┘                                             │                                         
   │                                                                  │                                         
   └──────────────────────────────────────────────────────────────────┘                                         
```
```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// ……
	
	// 如果 h 什么都没有，返回零值
	if h == nil || h.count == 0 {
		return unsafe.Pointer(&zeroVal[0])
	}
	
	// 写和读冲突
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	
	// 不同类型 key 使用的 hash 算法在编译期确定
	alg := t.key.alg
	
	// 计算哈希值，并且加入 hash0 引入随机性
	hash := alg.hash(key, uintptr(h.hash0))
	
	// 比如 B=5，那 m 就是31，二进制是全 1
	// 求 bucket num 时，将 hash 与 m 相与，
	// 达到 bucket num 由 hash 的低 8 位决定的效果
	m := uintptr(1)<<h.B - 1
	
	// b 就是 bucket 的地址
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	
	// oldbuckets 不为 nil，说明发生了扩容
	if c := h.oldbuckets; c != nil {
	    // 如果不是同 size 扩容（看后面扩容的内容）
	    // 对应条件 1 的解决方案
		if !h.sameSizeGrow() {
			// 新 bucket 数量是老的 2 倍
			m >>= 1
		}
		
		// 求出 key 在老的 map 中的 bucket 位置
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		
		// 如果 oldb 没有搬迁到新的 bucket
		// 那就在老的 bucket 中寻找
		if !evacuated(oldb) {
			b = oldb
		}
	}
	
	// 计算出高 8 位的 hash
	// 相当于右移 56 位，只取高8位
	top := uint8(hash >> (sys.PtrSize*8 - 8))
	
	// 增加一个 minTopHash
	if top < minTopHash {
		top += minTopHash
	}
	for {
	    // 遍历 8 个 bucket
		for i := uintptr(0); i < bucketCnt; i++ {
		    // tophash 不匹配，继续
			if b.tophash[i] != top {
				continue
			}
			// tophash 匹配，定位到 key 的位置
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			// key 是指针
			if t.indirectkey {
			    // 解引用
				k = *((*unsafe.Pointer)(k))
			}
			// 如果 key 相等
			if alg.equal(key, k) {
			    // 定位到 value 的位置
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				// value 解引用
				if t.indirectvalue {
					v = *((*unsafe.Pointer)(v))
				}
				return v
			}
		}
		
		// bucket 找完（还没找到），继续到 overflow bucket 里找
		b = b.overflow(t)
		// overflow bucket 也找完了，说明没有目标 key
		// 返回零值
		if b == nil {
			return unsafe.Pointer(&zeroVal[0])
		}
	}
}

```
```go
// key 定位公式
k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))

// value 定位公式
v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))

```

```go
b = b.overflow(t)
```

![57581783-fe5c2180-74ee-11e9-99c9-5a226216e1af.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1666171258390-4ce172a2-7559-43b1-9e8b-a10e046bbdc0.png#clientId=ucdd43824-96e5-4&errorMessage=unknown%20error&from=paste&height=1038&id=ubb15aedd&name=57581783-fe5c2180-74ee-11e9-99c9-5a226216e1af.png&originHeight=1038&originWidth=2248&originalType=binary&ratio=1&rotation=0&showTitle=false&size=115190&status=error&style=none&taskId=u3bf8c073-d013-421a-b609-5f13e47a445&title=&width=2248)
## 元素访问
## 赋值
## 遍历
## 删除
## 扩容
扩容发生在 mapassign 中

1. 是不是已经到了 load factor 的临界点，即元素个数 >= 桶个数 * 6.5，这时候说明大部分的桶可能都快满了，如果插入新元素，有大概率需要挂在 overflow 的桶上。
2. overflow 的桶是不是太多了，当 bucket 总数 < 2 ^ 15 时，如果 overflow 的 bucket 总数 >= bucket 的总数，那么我们认为 overflow 的桶太多了。当 bucket 总数 >= 2 ^ 15 时，那我们直接和 2 ^ 15 比较，overflow 的 bucket >= 2 ^ 15 时，即认为溢出桶太多了。为啥会导致这种情况呢？是因为我们对 map 一边插入，一边删除，会导致其中很多桶出现空洞，这样使得 bucket 使用率不高，值存储得比较稀疏。在查找时效率会下降。

两种情况官方采用了不同的解决方法:

- 针对 1，将 B + 1，进而 hmap 的 bucket 数组扩容一倍；
- 针对 2，通过移动 bucket 内容，使其倾向于紧密排列从而提高 bucket 利用率。
```go
func hashGrow(t *maptype, h *hmap) {
    // 如果已经超过了 load factor 的阈值，那么需要对 map 进行扩容，即 B = B + 1，bucket 总数会变为原来的二倍
    // 如果还没到阈值，那么只需要保持相同数量的 bucket，横向拍平就行了

    bigger := uint8(1)
    if !overLoadFactor(h.count+1, h.B) {
        bigger = 0
        h.flags |= sameSizeGrow
    }
    oldbuckets := h.buckets
    newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

    flags := h.flags &^ (iterator | oldIterator)
    if h.flags&iterator != 0 {
        flags |= oldIterator
    }

    // 提交扩容结果
    h.B += bigger
    h.flags = flags
    h.oldbuckets = oldbuckets
    h.buckets = newbuckets
    h.nevacuate = 0
    h.noverflow = 0

    if h.extra != nil && h.extra.overflow != nil {
        // 把当前的 overflow 赋值给 oldoverflow
        if h.extra.oldoverflow != nil {
            throw("oldoverflow is not nil")
        }
        h.extra.oldoverflow = h.extra.overflow
        h.extra.overflow = nil
    }
    if nextOverflow != nil {
        if h.extra == nil {
            h.extra = new(mapextra)
        }
        h.extra.nextOverflow = nextOverflow
    }

    // 实际的哈希表元素的拷贝工作是在 growWork 和 evacuate 中增量慢慢地进行的
}
```
```go
// makeBucketArray 为 map buckets 初始化底层数组
// 1<<b 是需要分配的最小数量 buckets
// dirtyalloc 要么是 nil，要么就是之前由 makeBucketArray 使用同样的 t 和 b 参数
// 分配的数组
// 如果 dirtyalloc 是 nil，那么就会分配一个新数组，否则会清空掉原来的 dirtyalloc
// 然后重用这部分内存作为新的底层数组
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
    // base = 1 << b
    base := bucketShift(b)
    nbuckets := base
    // 对于比较小的 b 来说，不太可能有 overflow buckets
    // 这里省掉一些计算消耗
    if b >= 4 {
        // Add on the estimated number of overflow buckets
        // required to insert the median number of elements
        // used with this value of b.
        nbuckets += bucketShift(b - 4)
        sz := t.bucket.size * nbuckets
        up := roundupsize(sz)
        if up != sz {
            nbuckets = up / t.bucket.size
        }
    }

    if dirtyalloc == nil {
        buckets = newarray(t.bucket, int(nbuckets))
    } else {
        // dirtyalloc 之前就是用上面的 newarray(t.bucket, int(nbuckets))
        // 生成的，所以是非空
        buckets = dirtyalloc
        size := t.bucket.size * nbuckets
        if t.bucket.kind&kindNoPointers == 0 {
            memclrHasPointers(buckets, size)
        } else {
            memclrNoHeapPointers(buckets, size)
        }
    }

    if base != nbuckets {
        // 我们预分配了一些 overflow buckets
        // 为了让追踪这些 overflow buckets 的成本最低
        // 我们这里约定，如果预分配的 overflow bucket 的 overflow 指针是 nil
        // 那么 there are more available by bumping the pointer.
        // 我们需要一个安全的非空指针来作为 last overflow bucket，直接用 buckets 就行了
        nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
        last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
        last.setoverflow(t, (*bmap)(buckets))
    }
    return buckets, nextOverflow
}
```
```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
    // 确保我们移动的 oldbucket 对应的是我们马上就要用到的那一个
    evacuate(t, h, bucket&h.oldbucketmask())

    // 如果还在 growing 状态，再多移动一个 oldbucket
    if h.growing() {
        evacuate(t, h, h.nevacuate)
    }
}

func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
    b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
    newbit := h.noldbuckets()
    if !evacuated(b) {
        // TODO: reuse overflow buckets instead of using new ones, if there
        // is no iterator using the old buckets.  (If !oldIterator.)

        // xy 包含的是移动的目标
        // x 表示新 bucket 数组的前(low)半部分
        // y 表示新 bucket 数组的后(high)半部分
        var xy [2]evacDst
        x := &xy[0]
        x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
        x.k = add(unsafe.Pointer(x.b), dataOffset)
        x.v = add(x.k, bucketCnt*uintptr(t.keysize))

        if !h.sameSizeGrow() {
            // 如果 map 大小(hmap.B)增大了，那么我们只计算 y
            // 否则 GC 可能会看到损坏的指针
            y := &xy[1]
            y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
            y.k = add(unsafe.Pointer(y.b), dataOffset)
            y.v = add(y.k, bucketCnt*uintptr(t.keysize))
        }

        for ; b != nil; b = b.overflow(t) {
            k := add(unsafe.Pointer(b), dataOffset)
            v := add(k, bucketCnt*uintptr(t.keysize))
            for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) {
                top := b.tophash[i]
                if top == empty {
                    b.tophash[i] = evacuatedEmpty
                    continue
                }
                if top < minTopHash {
                    throw("bad map state")
                }
                k2 := k
                if t.indirectkey {
                    k2 = *((*unsafe.Pointer)(k2))
                }
                var useY uint8
                if !h.sameSizeGrow() {
                    // 计算哈希，以判断我们的数据要转移到哪一部分的 bucket
                    // 可能是 x 部分，也可能是 y 部分
                    hash := t.key.alg.hash(k2, uintptr(h.hash0))
                    if h.flags&iterator != 0 && !t.reflexivekey && !t.key.alg.equal(k2, k2) {
                        // 为什么要加 reflexivekey 的判断，可以参考这里:
                        // https://go-review.googlesource.com/c/go/+/1480
                        // key != key，只有在 float 数的 NaN 时会出现
                        // 比如:
                        // n1 := math.NaN()
                        // n2 := math.NaN()
                        // fmt.Println(n1, n2)
                        // fmt.Println(n1 == n2)
                        // 这种情况下 n1 和 n2 的哈希值也完全不一样
                        // 这里官方表示这种情况是不可复现的
                        // 需要在 iterators 参与的情况下才能复现
                        // 但是对于这种 key 我们也可以随意对其目标进行发配
                        // 同时 tophash 对于 NaN 也没啥意义
                        // 还是按正常的情况下算一个随机的 tophash
                        // 然后公平地把这些 key 平均分布到各 bucket 就好
                        useY = top & 1 // 让这个 key 50% 概率去 Y 半区
                        top = tophash(hash)
                    } else {
                        // 这里写的比较 trick
                        // 比如当前有 8 个桶
                        // 那么如果 hash & 8 != 0
                        // 那么说明这个元素的 hash 这种形式
                        // xxx1xxx
                        // 而扩容后的 bucketMask 是
                        //    1111
                        // 所以实际上这个就是
                        // xxx1xxx & 1000 > 0
                        // 说明这个元素在扩容后一定会去下半区，即Y部分
                        // 所以就是 useY 了
                        if hash&newbit != 0 {
                            useY = 1
                        }
                    }
                }

                if evacuatedX+1 != evacuatedY {
                    throw("bad evacuatedN")
                }

                b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
                dst := &xy[useY]                 // 移动目标

                if dst.i == bucketCnt {
                    dst.b = h.newoverflow(t, dst.b)
                    dst.i = 0
                    dst.k = add(unsafe.Pointer(dst.b), dataOffset)
                    dst.v = add(dst.k, bucketCnt*uintptr(t.keysize))
                }
                dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
                if t.indirectkey {
                    *(*unsafe.Pointer)(dst.k) = k2 // 拷贝指针
                } else {
                    typedmemmove(t.key, dst.k, k) // 拷贝值
                }
                if t.indirectvalue {
                    *(*unsafe.Pointer)(dst.v) = *(*unsafe.Pointer)(v)
                } else {
                    typedmemmove(t.elem, dst.v, v)
                }
                dst.i++
                // These updates might push these pointers past the end of the
                // key or value arrays.  That's ok, as we have the overflow pointer
                // at the end of the bucket to protect against pointing past the
                // end of the bucket.
                dst.k = add(dst.k, uintptr(t.keysize))
                dst.v = add(dst.v, uintptr(t.valuesize))
            }
        }
        // Unlink the overflow buckets & clear key/value to help GC.
        if h.flags&oldIterator == 0 && t.bucket.kind&kindNoPointers == 0 {
            b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
            // Preserve b.tophash because the evacuation
            // state is maintained there.
            ptr := add(b, dataOffset)
            n := uintptr(t.bucketsize) - dataOffset
            memclrHasPointers(ptr, n)
        }
    }

    if oldbucket == h.nevacuate {
        advanceEvacuationMark(h, t, newbit)
    }
}

func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
    h.nevacuate++
    // Experiments suggest that 1024 is overkill by at least an order of magnitude.
    // Put it in there as a safeguard anyway, to ensure O(1) behavior.
    stop := h.nevacuate + 1024
    if stop > newbit {
        stop = newbit
    }
    for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
        h.nevacuate++
    }
    if h.nevacuate == newbit { // newbit == # of oldbuckets
        // 大小增长全部结束。释放老的 bucket array
        h.oldbuckets = nil
        // 同样可以丢弃老的 overflow buckets
        // 如果它们还被迭代器所引用的话
        // 迭代器会持有一份指向 slice 的指针
        if h.extra != nil {
            h.extra.oldoverflow = nil
        }
        h.flags &^= sameSizeGrow
    }
}
```
```go
                                                  ┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
                                                  │    0     │    1     │    2     │    3     │    4     │    5     │    6     │    7     │
                                                  └──────────┼──────────┴─────┬────┴──────────┴──────────┴──────────┴──────────┴──────────┘
                             ┌────────────────┐              │  tophash: 10   │                                                            
                             │     empty      │              │ lowbits: 1001  │                                                            
                             │                │◀─────┐       ├────────────────┤                                                            
                             ├────────────────┤      │       │     empty      │                                                            
                             │     empty      │      │       │                │                                                            
                             │                │      │       ├────────────────┤                                                            
                             ├────────────────┤      │       │     empty      │                                                            
                             │     empty      │      │       │                │                                                            
                    │        │                │      │       ├────────────────┤                                                            
                    │        ├────────────────┤      │       │  tophash: 23   │                                                            
                    │        │     empty      │      │       │ lowbits: 0001  │                                                            
                    │        │                │      │       ├────────────────┤                                                            
                    │        ├────────────────┤      │       │     empty      │                                                            
┌──────────┐        │        │     empty      │      │       │                │                                                            
│  before  │        │        │                │      │       ├────────────────┤                                                            
└──────────┘        │        ├────────────────┤      │       │  tophash: 100  │                                                            
                    │        │     empty      │      │       │ lowbits: 0001  │                                                            
                    │        │                │      │       ├────────────────┤                                                            
                    │        ├────────────────┤      │       │     empty      │                                                            
                    │        │  tophash: 15   │      │       │                │                                                            
                    │        │ lowbits: 0001  │      │       ├────────────────┤                                                            
                    │        ├────────────────┤      │       │     empty      │                                                            
                    │        │     empty      │      │       │                │                                                            
                    │        │                │      │       ├──────────┬─────┘                                                            
                    │        └────────────────┘      └───────│ overflow │                                                                  
                    │                                        └──────────┘                                                                  
                    │                                                                                                                      
                    │                                                                                                                      
                    │                                                                                                                      
                    │                                                                                                                      
                    │                                        ┌────────────────┐                                                            
                    │                                        │  tophash: 10   │                                                            
                    │                                        │ lowbits: 1001  │                                                            
                    │                                        ├────────────────┤                                                            
                    │                                        │  tophash: 23   │                                                            
                    │                                        │ lowbits: 0001  │                                                            
                    │                                        ├────────────────┤                                                            
                    │                                        │  tophash: 100  │                                                            
                    │                                        │ lowbits: 0001  │                                                            
                    │                                        ├────────────────┤                                                            
                    │                                        │  tophash: 15   │                                                            
                    │                                        │ lowbits: 0001  │                                                            
                    │                                        ├────────────────┤                                                            
┌──────────┐        │                                        │     empty      │                                                            
│  after   │        │                                        │                │                                                            
└──────────┘        │                                        ├────────────────┤                                                            
                    │                                        │     empty      │                                                            
                    │                                        │                │                                                            
                    │                                        ├────────────────┤                                                            
                    │                                        │     empty      │                                                            
                    │                                        │                │                                                            
                    │                                        ├────────────────┤                                                            
                    │                                        │     empty      │                                                            
                    │                                        │                │                                                            
                    │                             ┌──────────┼──────────┬─────┴────┬──────────┬──────────┬──────────┬──────────┬──────────┐
                    │                             │    0     │    1     │    2     │    3     │    4     │    5     │    6     │    7     │
                    │                             ├──────────┴──────────┴──────────┴──────────┴──────────┴──────────┴──────────┴──────────┤
                    │                             │                                                                                       │
                    │                             │                                                                                       │
                    │                             │                                                                                       │
                    │                             │                                                                                       │
                    │                             │                                                                                       │
                    │                             │                                                                                       │
                    │                             │                                                                                       │
                    │                             │◀─────────────────────────────         X part           ──────────────────────────────▶│
                    │                             │                                                                                       │
                    ▼                             │                                                                                       │
                                                  │                                                                                       │
                                                  │                                                                                       │
                                                  │                                                                                       │
```
sameSizeGrow之后，数据排列更紧凑
```go
┌──────────┐                                              ┌──────────┬───────────  
│    0     │                                              │    0     │     ▲       
├──────────┼───────────────┬────────────────┐             ├──────────┤     │       
│    1     │  tophash: 10  │   tophash:23   │      ┌─────▶│    1     │     │       
├──────────┤ lowbits: 1001 │ low bits: 0001 │──────┘      ├──────────┤     │       
│    2     ├───────────────┴────────────────┘             │    2     │     │       
├──────────┤       │                                      ├──────────┤             
│    3     │       │                                      │    3     │             
├──────────┤       │                                      ├──────────┤     X part  
│    4     │       │                                      │    4     │             
├──────────┤       │                                      ├──────────┤             
│    5     │       │                                      │    5     │     │       
├──────────┤       │                                      ├──────────┤     │       
│    6     │       │                                      │    6     │     │       
├──────────┤       │                                      ├──────────┤     │       
│    7     │       │                                      │    7     │     ▼       
└──────────┘       │                                      ├──────────┼───────────  
                   │                                      │    8     │     ▲       
                   │                                      ├──────────┤     │       
                   └─────────────────────────────────────▶│    9     │     │       
                                                          ├──────────┤     │       
                                                          │    10    │     │       
                                                          ├──────────┤             
                                                          │    11    │             
                                                          ├──────────┤     Y part  
                                                          │    12    │             
                                                          ├──────────┤             
                                                          │    13    │     │       
                                                          ├──────────┤     │       
                                                          │    14    │     │       
                                                          ├──────────┤     │       
                                                          │    15    │     ▼       
                                                          └──────────┴───────────  
```
桶数组增大以后，原来同一个桶的数据可以被分别移动到上半区和下半区
## indirectkey 和 indirectvalue
## overflow 处理
## 总结
## 拓展资料
## 使用map常见的两种错误
### 未初始化
和 slice 或者 Mutex、RWmutex 等 struct 类型不同，map 对象必须在使用之前初始化。如果不初始化就直接赋值的话，会出现 panic 异常，比如下面的例子，m 实例还没有初始化就直接进行操作会导致 panic（第 3 行）:
```go
func main() {
    var m map[int]int
    m[100] = 100
}

// fix 第二行
m := make(map[int]int)

// 从一个nil的map对象中获取值不会panic，而是会得到零值
func main() {
    var m map[int]int
    fmt.Println(m[100])
	// output: 0
}

// 将map作为struct的一个字段时，不要忘记初始化，所以下面代码会报错
type Counter struct {
    Website      string
    Start        time.Time
    PageCounters map[string]int
}

func main() {
    var c Counter
    c.Website = "baidu.com"
    c.PageCounters["/"]++
}
```
### 并发读写
Go 内建的 map 对象不是线程（goroutine）安全的，并发读写的时候运行时会有检查，遇到并发问题就会导致 panic。
```go
package main

func main() {
	var m = make(map[int]int, 10) // 初始化一个map
	go func() {
		for {
			m[1] = 1 //设置key
		}
	}()

	go func() {
		for {
			_ = m[2] //访问这个map
		}
	}()
	select {}
}
```
![{705501cb-3945-43c5-8b4e-0a1366396dc5}.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1666316751804-51a8b704-509a-4e1a-bef9-f9034af7a791.png#clientId=u7245521b-567c-4&from=paste&height=441&id=u51676905&name=%7B705501cb-3945-43c5-8b4e-0a1366396dc5%7D.png&originHeight=441&originWidth=650&originalType=binary&ratio=1&rotation=0&showTitle=false&size=43118&status=done&style=none&taskId=ue70d0b5d-3175-4327-b4fc-4974a6b90a1&title=&width=650)
## 如何实现线程安全的map类型
避免 map 并发读写 panic 的方式之一就是加锁，考虑到读写性能，可以使用读写锁提供性能。
### 加读写锁：扩展 map，支持并发读写
```go

type RWMap struct { // 一个读写锁保护的线程安全的map
    sync.RWMutex // 读写锁保护下面的map字段
    m map[int]int
}
// 新建一个RWMap
func NewRWMap(n int) *RWMap {
    return &RWMap{
        m: make(map[int]int, n),
    }
}
func (m *RWMap) Get(k int) (int, bool) { //从map中读取一个值
    m.RLock()
    defer m.RUnlock()
    v, existed := m.m[k] // 在锁的保护下从map中读取
    return v, existed
}

func (m *RWMap) Set(k int, v int) { // 设置一个键值对
    m.Lock()              // 锁保护
    defer m.Unlock()
    m.m[k] = v
}

func (m *RWMap) Delete(k int) { //删除一个键
    m.Lock()                   // 锁保护
    defer m.Unlock()
    delete(m.m, k)
}

func (m *RWMap) Len() int { // map的长度
    m.RLock()   // 锁保护
    defer m.RUnlock()
    return len(m.m)
}

func (m *RWMap) Each(f func(k, v int) bool) { // 遍历map
    m.RLock()             //遍历期间一直持有读锁
    defer m.RUnlock()

    for k, v := range m.m {
        if !f(k, v) {
            return
        }
    }
}

```
### 分片加锁：更高效的并发 map
虽然使用读写锁可以提供线程安全的 map，但是在大量并发读写的情况下，锁的竞争会非常激烈。锁是性能下降的万恶之源之一。
在并发编程中，需要尽量减少锁的使用。一些单线程单进程的应用（比如 Redis 等），基本上不需要使用锁去解决并发线程访问的问题，所以可以取得很高的性能。但是对于 Go 开发的应用程序来说，并发是常用的一个特性，需要尽量减少锁的粒度和锁的持有时间。可以优化业务处理的代码，以此来减少锁的持有时间，比如将串行的操作变成并行的子任务执行。
减少锁的粒度。减少锁的粒度常用的方法就是分片（Shard），将一把锁分成几把锁，每个锁控制一个分片。Go 比较知名的分片并发 map 的实现是[orcaman/concurrent-map](https://github.com/orcaman/concurrent-map)。
它默认采用 32 个分片，GetShard 是一个关键的方法，能够根据 key 计算出分片索引。
```go
	 var SHARD_COUNT = 32
  
    // 分成SHARD_COUNT个分片的map
  type ConcurrentMap []*ConcurrentMapShared
  
  // 通过RWMutex保护的线程安全的分片，包含一个map
  type ConcurrentMapShared struct {
    items        map[string]interface{}
    sync.RWMutex // Read Write mutex, guards access to internal map.
  }
  
  // 创建并发map
  func New() ConcurrentMap {
    m := make(ConcurrentMap, SHARD_COUNT)
    for i := 0; i < SHARD_COUNT; i++ {
      m[i] = &ConcurrentMapShared{items: make(map[string]interface{})}
    }
    return m
  }
  

  // 根据key计算分片索引
  func (m ConcurrentMap) GetShard(key string) *ConcurrentMapShared {
    return m[uint(fnv32(key))%uint(SHARD_COUNT)]
  }
```
增加或者查询的时候，首先根据分片索引得到分片对象，然后对分片对象加锁进行操作：
```go
func (m ConcurrentMap) Set(key string, value interface{}) {
    // 根据key计算出对应的分片
    shard := m.GetShard(key)
    shard.Lock() //对这个分片加锁，执行业务操作
    shard.items[key] = value
    shard.Unlock()
}

func (m ConcurrentMap) Get(key string) (interface{}, bool) {
    // 根据key计算出对应的分片
    shard := m.GetShard(key)
    shard.RLock()
    // 从这个分片读取key的值
    val, ok := shard.items[key]
    shard.RUnlock()
    return val, ok
}
```
加锁和分片加锁这两种方案都比较常用，如果是追求更高的性能，显然是分片加锁更好，因为它可以降低锁的粒度，进而提高访问此 map 对象的吞吐。如果并发性能要求不是那么高的场景，简单加锁方式更简单。
# Sync.Map
 	Go 语言原生 map 并不是线程安全的，对它进行并发读写操作的时候，需要加锁。而 sync.map 则是一种并发安全的 map，在 Go 1.9 引入。  
> sync.map 是线程安全的，读取，插入，删除也都保持着常数级的时间复杂度。
sync.map 的零值是有效的，并且零值是一个空的 map。在第一次使用之后，不允许被拷贝。  

 sync.Map 并不是用来替换内建的 map 类型的，它只能被应用在一些特殊的场景里。[官方的文档中](https://pkg.go.dev/sync#Map)指出，在以下两个场景中使用 sync.Map，会比使用 map+RWMutex 的方式，性能要好得多：

   1. 只会增长的缓存系统中，一个 key 只写入一次而被读很多次；
   2. 多个 goroutine 为不相交的键集读、写和重写键值对。
## 优化点

- **空间换时间。通过冗余的两个数据结构（只读的 read 字段、可写的 dirty），来减少加锁对性能的影响。**
- **对只读字段（read）的操作不需要加锁。优先从 read 字段读取、更新、删除，因为对 read 字段的读取不需要锁。**
- **动态调整。miss 次数多了之后，将 dirty 数据提升为 read，避免总是从 dirty 中加锁读取。**
- **double-checking。加锁之后先还要再检查 read 字段，确定真的不存在才操作 dirty 字段。**
- **延迟删除。删除一个键值只是打标记，只有在提升 dirty 字段为 read 字段的时候才清理删除的数据。**
## 简单使用
```go
package main

import (
	"fmt"
	"sync"
)

func main()  {
	var m sync.Map
	// 1. 写入
	m.Store("Jordan", 50)
	m.Store("James", 38)

	// 2. 读取
	age, _ := m.Load("James")
	fmt.Println(age.(int))

	// 3. 遍历
	m.Range(func(key, value interface{}) bool {
		name := key.(string)
		age := value.(int)
		fmt.Println(name, age)
		return true
	})

	// 5. 读取或写入（已存在的）
	m.LoadOrStore("Jordan", 55)
	age, _ = m.Load("Jordan")
	fmt.Println(age)

	// 6. 读取或写入（不存在的）
	m.LoadOrStore("James", 40)
	age, _ = m.Load("James")
	fmt.Println(age)
}

```

   1. 写入两个 k-v 对；
   2. 使用 Load 方法读取其中的一个 key；
   3. 遍历所有的 k-v 对，并打印出来；
   4. 删除其中的一个 key，再读这个 key，得到的就是 nil；
   5. 使用 LoadOrStore，尝试读取或写入 "Jordan"，因为这个 key 已经存在，因此写入不成功，并且读出原值。
   6. 使用LoadOrStore，尝试读取或写入 "James"，因为这个key不存在，因此写入成功，读取值得时候返回刚才store的值

![{46bc616a-dde2-4027-b380-049b168b3f54}.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1666315753998-089d936a-fed5-4498-8db1-a15261d02786.png#clientId=u7245521b-567c-4&from=paste&height=172&id=u9c89f2e0&name=%7B46bc616a-dde2-4027-b380-049b168b3f54%7D.png&originHeight=172&originWidth=696&originalType=binary&ratio=1&rotation=0&showTitle=false&size=10771&status=done&style=none&taskId=u576b324c-3b02-4e55-b44c-28b675f12f6&title=&width=696)
**sync.map 适用于读多写少的场景。对于写多的场景，会导致 read map 缓存失效，需要加锁，导致冲突变多；而且由于未命中 read map 次数过多，导致 dirty map 提升为 read map，这是一个 O(N) 的操作，会进一步降低性能。  **
## 数据结构
```go
type Map struct {
    mu Mutex
    // 基本上你可以把它看成一个安全的只读的map
    // 它包含的元素其实也是通过原子操作更新的，但是已删除的entry就需要加锁操作了
    read atomic.Value // readOnly

    // 包含需要加锁才能访问的元素
    // 包括所有在read字段中但未被expunged（删除）的元素以及新加的元素
    dirty map[interface{}]*entry

    // 记录从read中读取miss的次数，一旦miss数和dirty长度一样了，就会把dirty提升为read，并把dirty置空
    misses int
}

type readOnly struct {
    m       map[interface{}]*entry
    amended bool // 当dirty中包含read没有的数据时为true，比如新增一条数据
}
```
互斥量 mu 保护 read 和 dirty。
read 是 atomic.Value 类型，可以并发地读。但如果需要更新 read，则需要加锁保护。对于 read 中存储的 entry 字段，可能会被并发地 CAS 更新。但是如果要更新一个之前已被删除的 entry，则需要先将其状态从 expunged 改为 nil，再拷贝到 dirty 中，然后再更新。
dirty 是一个非线程安全的原始 map。包含新写入的 key，并且包含 read 中的所有未被删除的 key。这样，可以快速地将 dirty 提升为 read 对外提供服务。如果 dirty 为 nil，那么下一次写入时，会新建一个新的 dirty，这个初始的 dirty 是 read 的一个拷贝，但除掉了其中已被删除的 key。
每当从 read 中读取失败，都会将 misses 的计数值加 1，当加到一定阈值以后，需要将 dirty 提升为 read，以期减少 miss 的情形。
> read map 和 dirty map 的存储方式是不一致的。
	前者使用 atomic.Value，后者只是单纯的使用 map。
	原因是 read map 使用 lock free 操作，必须保证 load/store 的原子性；而 dirty map 的 load+store 操作是由 lock（就是 mu）来保护的。  

```go
// entry代表一个值
type entry struct {
    p unsafe.Pointer // *interface{}
}
```
![20200504093007.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1666325361058-5ae627b8-d337-4399-a00a-92814149dbfa.png#clientId=u7245521b-567c-4&from=paste&height=354&id=ucf44f778&name=20200504093007.png&originHeight=354&originWidth=990&originalType=binary&ratio=1&rotation=0&showTitle=false&size=25689&status=done&style=none&taskId=uf47b92e9-9a1b-49bc-bf17-42942048cdd&title=&width=990)
当 p == nil 时，说明这个键值对已被删除，并且 m.dirty == nil，或 m.dirty[k] 指向该 entry。
当 p == expunged 时，说明这条键值对已被删除，并且 m.dirty != nil，且 m.dirty 中没有这个 key。
其他情况，p 指向一个正常的值，表示实际 interface{} 的地址，并且被记录在 m.read.m[key] 中。如果这时 m.dirty 不为 nil，那么它也被记录在 m.dirty[key] 中。两者实际上指向的是同一个值。
当删除 key 时，并不实际删除。一个 entry 可以通过原子地（CAS 操作）设置 p 为 nil 被删除。如果之后创建 m.dirty，nil 又会被原子地设置为 expunged，且不会拷贝到 dirty 中。
如果 p 不为 expunged，和 entry 相关联的这个 value 可以被原子地更新；如果 p == expunged，那么仅当它初次被设置到 m.dirty 之后，才可以被更新。
![20200505120255.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1666325414409-3ce6e5e8-ddb2-44fd-bb56-d98610704022.png#clientId=u7245521b-567c-4&from=paste&height=652&id=ud3d4b34f&name=20200505120255.png&originHeight=652&originWidth=1512&originalType=binary&ratio=1&rotation=0&showTitle=false&size=68354&status=done&style=none&taskId=u4f7e7830-c2dd-47ae-9dc9-8436bf0fac3&title=&width=1512)
## Store
```go
// expunged是用来标识此项已经删掉的指针
// 当map中的一个项目被删除了，只是把它的值标记为expunged，以后才有机会真正删除此项
var expunged = unsafe.Pointer(new(interface{}))
```
```go
// Store sets the value for a key.
func (m *Map) Store(key, value interface{}) {
	// 如果 read map 中存在该 key  则尝试直接更改(由于修改的是 entry 内部的 pointer，因此 dirty map 也可见)
	read, _ := m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok && e.tryStore(&value) {
		return
	}

	m.mu.Lock()
	read, _ = m.read.Load().(readOnly)
	if e, ok := read.m[key]; ok {
		if e.unexpungeLocked() {
			// 如果 read map 中存在该 key，但 p == expunged，则说明 m.dirty != nil 并且 m.dirty 中不存在该 key 值 此时:
			//    a. 将 p 的状态由 expunged  更改为 nil
			//    b. dirty map 插入 key
			m.dirty[key] = e
		}
		// 更新 entry.p = value (read map 和 dirty map 指向同一个 entry)
		e.storeLocked(&value)
	} else if e, ok := m.dirty[key]; ok {
		// 如果 read map 中不存在该 key，但 dirty map 中存在该 key，直接写入更新 entry(read map 中仍然没有这个 key)
		e.storeLocked(&value)
	} else {
		// 如果 read map 和 dirty map 中都不存在该 key，则：
		//	  a. 如果 dirty map 为空，则需要创建 dirty map，并从 read map 中拷贝未删除的元素到新创建的 dirty map
		//    b. 更新 amended 字段，标识 dirty map 中存在 read map 中没有的 key
		//    c. 将 kv 写入 dirty map 中，read 不变
		if !read.amended {
		    // 到这里就意味着，当前的 key 是第一次被加到 dirty map 中。
			// store 之前先判断一下 dirty map 是否为空，如果为空，就把 read map 浅拷贝一次。
			m.dirtyLocked()
			m.read.Store(readOnly{m: read.m, amended: true})
		}
		// 写入新 key，在 dirty 中存储 value
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
}

```
整体流程：

1. 如果在 read 里能够找到待存储的 key，并且对应的 entry 的 p 值不为 expunged，也就是没被删除时，直接更新对应的 entry 即可。
2. 第一步没有成功：要么 read 中没有这个 key，要么 key 被标记为删除。则先加锁，再进行后续的操作。
3. 再次在 read 中查找是否存在这个 key，也就是 double check 一下，这也是 lock-free 编程里的常见套路。如果 read 中存在该 key，但 p == expunged，说明 m.dirty != nil 并且 m.dirty 中不存在该 key 值 此时: a. 将 p 的状态由 expunged 更改为 nil；b. dirty map 插入 key。然后，直接更新对应的 value。
4. 如果 read 中没有此 key，那就查看 dirty 中是否有此 key，如果有，则直接更新对应的 value，这时 read 中还是没有此 key。
5. 最后一步，如果 read 和 dirty 中都不存在该 key，则：a. 如果 dirty 为空，则需要创建 dirty，并从 read 中拷贝未被删除的元素；b. 更新 amended 字段，标识 dirty map 中存在 read map 中没有的 key；c. 将 k-v 写入 dirty map 中，read.m 不变。最后，更新此 key 对应的 value。
```go

func (m *Map) dirtyLocked() {
    if m.dirty != nil { // 如果dirty字段已经存在，不需要创建了
        return
    }

    read, _ := m.read.Load().(readOnly) // 获取read字段
    m.dirty = make(map[interface{}]*entry, len(read.m))
    for k, e := range read.m { // 遍历read字段
        if !e.tryExpungeLocked() { // 把非punged的键值对复制到dirty中
            m.dirty[k] = e
        }
    }
}

// tryStore 在 Store 函数最开始的时候就会调用，是比较常见的 for 循环加 CAS 操作，尝试更新 entry，让 p 指向新的值。
// 如果 entry 没被删，tryStore 存储值到 entry 中。如果 p == expunged，即 entry 被删，那么返回 false。
func (e *entry) tryStore(i *interface{}) bool {
	for {
		p := atomic.LoadPointer(&e.p)
		if p == expunged {
			return false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
			return true
		}
	}
}


// unexpungeLocked 函数确保了 entry 没有被标记成已被清除。
// 如果 entry 先前被清除过了，那么在 mutex 解锁之前，它一定要被加入到 dirty map 中
func (e *entry) unexpungeLocked() (wasExpunged bool) {
	return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

```
## Load
```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	// 如果没在 read 中找到，并且 amended 为 true，即 dirty 中存在 read 中没有的 key
	if !ok && read.amended {
		m.mu.Lock() // dirty map 不是线程安全的，所以需要加上互斥锁
		// double check。避免在上锁的过程中 dirty map 提升为 read map。
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		// 仍然没有在 read 中找到这个 key，并且 amended 为 true
		if !ok && read.amended {
			e, ok = m.dirty[key] // 从 dirty 中找
			// 不管 dirty 中有没有找到，都要"记一笔"，因为在 dirty 提升为 read 之前，都会进入这条路径
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok { // 如果没找到，返回空，false
		return nil, false
	}
	return e.load()
}

```
处理路径分为 fast path 和 slow path，整体流程如下：

1. 首先是 fast path，直接在 read 中找，如果找到了直接调用 entry 的 load 方法，取出其中的值。
2. 如果 read 中没有这个 key，且 amended 为 fase，说明 dirty 为空，那直接返回 空和 false。
3. 如果 read 中没有这个 key，且 amended 为 true，说明 dirty 中可能存在我们要找的 key。当然要先上锁，再尝试去 dirty 中查找。在这之前，仍然有一个 double check 的操作。若还是没有在 read 中找到，那么就从 dirty 中找。不管 dirty 中有没有找到，都要"记一笔"，因为在 dirty 被提升为 read 之前，都会进入这条路径
```go

func (m *Map) missLocked() {
    m.misses++ // misses计数加一
    if m.misses < len(m.dirty) { // 如果没达到阈值(dirty字段的长度),返回
        return
    }
    m.read.Store(readOnly{m: m.dirty}) //把dirty字段的内存提升为read字段
    m.dirty = nil // 清空dirty
    m.misses = 0  // misses数重置为0
}
```
 	直接将 misses 的值加 1，表示一次未命中，如果 misses 值小于 m.dirty 的长度，就直接返回。否则，将 m.dirty 晋升为 read，并清空 dirty，清空 misses 计数值。这样，之前一段时间新加入的 key 都会进入到 read 中，从而能够提升 read 的命中率。  
```go
func (e *entry) load() (value interface{}, ok bool) {
	p := atomic.LoadPointer(&e.p)
	if p == nil || p == expunged {
		return nil, false
	}
	return *(*interface{})(p), true
}

```
 	对于 nil 和 expunged 状态的 entry，直接返回 ok=false；否则，将 p 转成 interface{} 返回。  
## Delete
```go
// Delete deletes the value for a key.
func (m *Map) Delete(key interface{}) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	// 如果 read 中没有这个 key，且 dirty map 不为空
	if !ok && read.amended {
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			delete(m.dirty, key) // 直接从 dirty 中删除这个 key
		}
		m.mu.Unlock()
	}
	if ok {
		e.delete() // 如果在 read 中找到了这个 key，将 p 置为 nil
	}
}

```
 	read 中找到这个 key，并且 dirty 不为空，那么就要操作 dirty 了，操作之前，还是要先上锁。然后进行 double check，如果仍然没有在 read 里找到此 key，则从 dirty 中删掉这个 key。但不是真正地从 dirty 中删除，而是更新 entry 的状态。  
```go
func (e *entry) delete() (hadValue bool) {
	for {
		p := atomic.LoadPointer(&e.p)
		if p == nil || p == expunged {
			return false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, nil) {
			return true
		}
	}
}

```
它真正做的事情是将正常状态（指向一个 interface{}）的 p 设置成 nil。没有设置成 expunged 的原因是，当 p 为 expunged 时，表示它已经不在 dirty 中了。这是 p 的状态机决定的，在 tryExpungeLocked 函数中，会将 nil 原子地设置成 expunged。
tryExpungeLocked 是在新创建 dirty 时调用的，会将已被删除的 entry.p 从 nil 改成 expunged，这个 entry 就不会写入 dirty 了。
```go
func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := atomic.LoadPointer(&e.p)
	for p == nil {
		// 如果原来是 nil，说明原 key 已被删除，则将其转为 expunged。
		if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
			return true
		}
		p = atomic.LoadPointer(&e.p)
	}
	return p == expunged
}

```
 	key 同时存在于 read 和 dirty 中时，删除只是做了一个标记，将 p 置为 nil；而如果仅在 dirty 中含有这个 key 时，会直接删除这个 key。原因在于，若两者都存在这个 key，仅做标记删除，可以在下次查找这个 key 时，命中 read，提升效率。若只有在 dirty 中存在时，read 起不到“缓存”的作用，直接删除。  
## Range
```go
f func(key, value interface{}) bool
```
```go
func (m *Map) Range(f func(key, value interface{}) bool) {
	read, _ := m.read.Load().(readOnly)
	if read.amended {
		m.mu.Lock()
		read, _ = m.read.Load().(readOnly)
		if read.amended {
			read = readOnly{m: m.dirty}
			m.read.Store(read)
			m.dirty = nil
			m.misses = 0
		}
		m.mu.Unlock()
	}

	for k, e := range read.m {
		v, ok := e.load()
		if !ok {
			continue
		}
		if !f(k, v) {
			break
		}
	}
}

```
当 amended 为 true 时，说明 dirty 中含有 read 中没有的 key，因为 Range 会遍历所有的 key，是一个 O(n) 操作。将 dirty 提升为 read，会将开销分摊开来，所以这里直接就提升了。之后，遍历 read，取出 entry 中的值，调用 f(k, v)。
## 总结

- sync.map 是线程安全的，读取，插入，删除也都保持着常数级的时间复杂度。
- 通过读写分离，降低锁时间来提高效率，适用于读多写少的场景。
- Range 操作需要提供一个函数，参数是 k,v，返回值是一个布尔值：f func(key, value interface{}) bool。
- 调用 Load 或 LoadOrStore 函数时，如果在 read 中没有找到 key，则会将 misses 值原子地增加 1，当 misses 增加到和 dirty 的长度相等时，会将 dirty 提升为 read。以期减少“读 miss”。
- 新写入的 key 会保存到 dirty 中，如果这时 dirty 为 nil，就会先新创建一个 dirty，并将 read 中未被删除的元素拷贝到 dirty。
- 当 dirty 为 nil 的时候，read 就代表 map 所有的数据；当 dirty 不为 nil 的时候，dirty 才代表 map 所有的数据。
## ![a80408a137b13f934b0dd6f2b6c5cc03.webp](https://cdn.nlark.com/yuque/0/2022/webp/1607733/1666326146788-6293cf6c-7f9e-4f74-9184-960c80c338ab.webp#clientId=u7245521b-567c-4&from=paste&height=1771&id=u3bb6e5f6&name=a80408a137b13f934b0dd6f2b6c5cc03.webp&originHeight=1771&originWidth=2250&originalType=binary&ratio=1&rotation=0&showTitle=false&size=190592&status=done&style=none&taskId=ufb44227f-3dfa-48fe-8a5b-ab8c5f19c68&title=&width=2250)
## 拓展资料

- [如何设计并实现一个线程安全的 Map ？(上篇)](https://halfrost.com/go_map_chapter_one/)
- [如何设计并实现一个线程安全的 Map ？(下篇)](https://halfrost.com/go_map_chapter_two/)
- [扩展go sync.map的length和delete方法](https://xiaorui.cc/archives/4972)
- [sync.map为什么没有len方法的issue](https://github.com/golang/go/issues/20680)
- [带有过期功能的key map](https://github.com/zekroTJA/timedmap)
- [使用红黑树实现的key有序的treemap](https://pkg.go.dev/github.com/emirpasic/gods/maps/treemap)

