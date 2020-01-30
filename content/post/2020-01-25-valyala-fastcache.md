+++
title="valyala/fastcache 源码分析"
tags=["golang"]
categories=["源码分析"]
date="2020-01-25T12:49:11+08:00"
+++

如果你是一个追求性能的golanger，那么你肯定听说过fasthttp、quicktemplate、fastjson，而这些高性能库都是出自valyala大神之手，今天就来学习一下v神的多线程缓存库[fastcache](https://github.com/victoriametrics/fastcache)。

## 使用

fastcache提供了非常简单的接口，下面是一个用例，可以注意到所有的缓存操作都不返回error，这是非常合理的设计，缓存本身就是一种可以丢数据的东西，其所有的操作都应该始终尽力而为的模式。

```go
func TestKVOperation(t *testing.T) {
	c := fastcache.New(512 * 1024 * 1024)

	// 基本使用
	c.Set([]byte("key1"),  []byte("value1"))
	spew.Dump(c.Get(nil, []byte("key1")))

	// 支持指定value的输出位置，令用户可以选择性复用[]byte
	c.Set([]byte("key2"),  []byte("value2"))
	dst := make([]byte, 0, 6)
	dst = c.Get(dst, []byte("key2"))
	spew.Dump(dst)

	// 对于len(key) + len(value) > (65536 - 4)的缓存项目需要使用另外的API
	// 当然，较小的kv也可以使用SetBig/GetBig来缓存
	c.SetBig([]byte("BigKey1"), []byte("BigValue1"))
	spew.Dump(c.GetBig(nil, []byte("BigKey1")))

	// 没有命中缓存时，可以这样判断
	dst2 := []byte("something already in dst []byte")
	dst2OldLen := len(dst2)
	dst2 = c.Get(dst2, []byte("key not cached"))
	if len(dst2) == dst2OldLen {
		fmt.Println("cache misses")
	}

	// 判断一个值是否在cache中
	c.Set([]byte("key3"),  []byte{})
	fmt.Println(c.Has([]byte("key3")))

	// 删除一个值
	c.Del([]byte("key3"))
	fmt.Println(c.Has([]byte("key3")))

	// 清空缓存，使用reset方法可以复用已经分配的内存
	c.Reset()
}
```

fastcache暴漏了内部指标，为我们改进代码提供数据支持。并且其指标没有强依赖prometheus-client，而是以单独的结构提提供，为用户提供了更大的灵活性。

```go
func TestStatus(t *testing.T) {
	c := fastcache.New(512 * 1024 * 1024)
	c.Set([]byte("key1"),  []byte("value1"))
	c.Set([]byte("key2"),  []byte("value2"))
	c.SetBig([]byte("BigKey1"), []byte("BigValue1"))
	c.Get(nil, []byte("key not existed"))
	c.Get(nil, []byte("key1"))
	c.GetBig(nil, []byte("key2"))

	s := &fastcache.Stats{}
	c.UpdateStats(s)
	spew.Dump(s)
}
```

fastcache可以将缓存持久化到硬盘：

```go
func TestDumpLoad(t *testing.T) {
	cMem := fastcache.New(512 * 1024 * 1024)
	cMem.Set([]byte("key1"),  []byte("value1"))
	cMem.Set([]byte("key2"),  []byte("value2"))
	cMem.SetBig([]byte("BigKey1"), []byte("BigValue1"))

	tmpDir, err := ioutil.TempDir("", "fastcache")
	if err != nil {
		log.Fatal(err)
	}

	if err := cMem.SaveToFile(tmpDir); err != nil {
		log.Fatal(err)
	}

	cFile := fastcache.LoadFromFileOrNew(tmpDir, 512 * 1024 * 1024)
	spew.Dump(cFile.Get(nil, []byte("key1")))
	spew.Dump(cFile.Get(nil, []byte("key2")))
	spew.Dump(cFile.GetBig(nil, []byte("BigKey1")))
}

// /tmp/fastcache022053982/
// ├── data.0.bin
// └── metadata.bin
```

## 数据结构

先来看fastcache的数据结构，每个cache实例中包含512个桶，每个桶中包含若干64KiB的chunk，chunk的数量由初始化时配置的maxBytes决定，maxBytes不小于32MiB，这是为了保证512个bucket中都至少有1个chunk。

```go
const (
	bucketsCount          = 512
	chunkSize             = 64 * 1024
	bucketSizeBits        = 40
	genSizeBits           = 64 - bucketSizeBits
	maxGen                = 1<<genSizeBits - 1
	maxBucketSize  uint64 = 1 << bucketSizeBits
)

// 分桶减小锁粒度
type Cache struct {
	buckets [bucketsCount]bucket
}

type bucket struct {
	mu sync.RWMutex

	// chunks组成的一个ring buffer，Key和value都会存储到chunk中
	// 使用ring buffer的一个好处是，不需要复杂的驱逐策略，驱逐直接写入Set过程中
	// 不需要单独加锁，生产环境中使用LRU，常常会在驱逐时造成长时间加锁，导致缓存效率底下
	chunks [][]byte

	// 保存hash(key) -> <gen(高14位), idx(低40位)>的映射
	m map[uint64]uint64
  
	// idx 保存已经写入该桶的字节数，用于计算下一个chunk的写入位置
	// 当ring buffer被写满后，idx会重0开始
	idx uint64

	// gen ring buffer的generation，从1开始，递增到(1<<14)后，重新回到1开始
	gen uint64
}
```

## New

```go
func New(maxBytes int) *Cache {
	if maxBytes <= 0 {
		panic(fmt.Errorf("maxBytes must be greater than 0; got %d", maxBytes))
	}
	// 预先将所有桶初始化好，以保证不需要用使用全局锁来创建桶
	var c Cache
	maxBucketBytes := uint64((maxBytes + bucketsCount - 1) / bucketsCount)
	for i := range c.buckets[:] {
		c.buckets[i].Init(maxBucketBytes)
	}
	return &c
}

func (b *bucket) Init(maxBytes uint64) {
	if maxBytes == 0 {
		panic(fmt.Errorf("maxBytes cannot be zero"))
	}
	if maxBytes >= maxBucketSize {
		panic(fmt.Errorf("too big maxBytes=%d; should be smaller than %d", maxBytes, maxBucketSize))
	}
	maxChunks := (maxBytes + chunkSize - 1) / chunkSize
	b.chunks = make([][]byte, maxChunks)
	b.m = make(map[uint64]uint64)
	b.Reset()
}

```

## Set & Get & Del & Clean

fastcache对不同大小的缓存项目有不同的存储策略，我们先来看小于64KiB的Set：

```go
// 首先通过（号称最快的）哈希算法xxhash算key的hash来选桶
// 后由具体的桶来执行set操作，这样可以减少锁的粒度，提升并发性能
func (c *Cache) Set(k, v []byte) {
	h := xxhash.Sum64(k)
	idx := h % bucketsCount
	c.buckets[idx].Set(k, v, h)
}

func (b *bucket) Set(k, v []byte, h uint64) {
	// 所有的指标更新都是通过无锁的atomic包实现（和promehteus client是一样的）
	setCalls := atomic.AddUint64(&b.setCalls, 1)
  
	// 当set超过一定数量时，就删除ring buffer中的旧generation chunks
	if setCalls%(1<<14) == 0 {
		b.Clean()
	}
  
	// 每个缓存项的格式如下，每个chunk中可以包换多个缓存项：
	// 但若一个chunk放不下一个这样的结构时，会跳到下一个块写（会存在一些内存浪费）
	// +------------+--------------+------------------+-------------------+
	// | Key Length | Value Length |       Key        |       Value       |
	// +------------+--------------+------------------+-------------------+
	// 下面的代码保证key、value、key length、value length可以放在一个chunk中。

	if len(k) >= (1<<16) || len(v) >= (1<<16) {
		// Too big key or value - its length cannot be encoded
		// with 2 bytes (see below). Skip the entry.
		return
	}
	// maxSubvalueLen中的4
	var kvLenBuf [4]byte
	kvLenBuf[0] = byte(uint16(len(k)) >> 8)
	kvLenBuf[1] = byte(len(k))
	kvLenBuf[2] = byte(uint16(len(v)) >> 8)
	kvLenBuf[3] = byte(len(v))
	kvLen := uint64(len(kvLenBuf) + len(k) + len(v))
	if kvLen >= chunkSize {
		// Do not store too big keys and values, since they do not
		// fit a chunk.
		return
	}
  
	b.mu.Lock()
	idx := b.idx
	idxNew := idx + kvLen
	chunkIdx := idx / chunkSize
	chunkIdxNew := idxNew / chunkSize
	if chunkIdxNew > chunkIdx {
		if chunkIdxNew >= uint64(len(b.chunks)) {
			idx = 0
			idxNew = kvLen
			chunkIdx = 0
			b.gen++
			// 注意如果b.gen从0开始，则无法判断是否一次gen递增过程是否完成
			if b.gen&((1<<genSizeBits)-1) == 0 {
				b.gen++
			}
		} else {
			idx = chunkIdxNew * chunkSize
			idxNew = idx + kvLen
			chunkIdx = chunkIdxNew
		}
		b.chunks[chunkIdx] = b.chunks[chunkIdx][:0]
	}
	chunk := b.chunks[chunkIdx]
	if chunk == nil {
		// getChunk从预先分配的内存池中拿缓存块，这部分在下面介绍
		chunk = getChunk()
		chunk = chunk[:0]
	}
	// 写入KV，更新索引，更新idx
	chunk = append(chunk, kvLenBuf[:]...)
	chunk = append(chunk, k...)
	chunk = append(chunk, v...)
	b.chunks[chunkIdx] = chunk
	b.m[h] = idx | (b.gen << bucketSizeBits)
	b.idx = idxNew
	b.mu.Unlock()
}
```

接下来我们看如何Get

```go
// 和set相同的模式分桶，减小锁粒度;
// 最后的拿到的缓存项会放入dst，为使用者提供复用内存的机会；
func (c *Cache) Get(dst, k []byte) []byte {
	h := xxhash.Sum64(k)
	idx := h % bucketsCount
	dst, _ = c.buckets[idx].Get(dst, k, h, true)
	return dst
}

// 返回dst和是否找到，returnDst决定是否将找到的value写入dst
// 通过这个returnDst开关，这个函数就能作为Has来使用了，这也是Has的实现方法
func (b *bucket) Get(dst, k []byte, h uint64, returnDst bool) ([]byte, bool) {
	found := false
	b.mu.RLock()
	v := b.m[h]
	bGen := b.gen & ((1 << genSizeBits) - 1)
	if v > 0 {
		gen := v >> bucketSizeBits // 要Get的缓存项的gen
		idx := v & ((1 << bucketSizeBits) - 1) // 要Get的缓存项的idx 
		// 注意这里到的gen判断：
		// 若ring buffer和别取的值在同一个gen中，则有：
		//        idx              b.idx
		//         |                 |
		// +-------v-----------------v-------+
		//              same gen
		// 若ring buffer已经是一个新的gen，但旧的值没有被完全覆写，即gen+1==bGen，则有：
		//       b.idx              idx
		//         |                 |
		// +-------v--------.........v.......+
		//      new gen           old gen
		// 注意，b.idx保存的是目前的总字节数，而go的下标从0开始，
		// 所以b.idx指向的是下一个写入位置，所以还没有被写入，所以要包含相等的情况；
		// 另外，gen是通过模运算得来，所以要之前的gen已经是最大的maxGen，而当前gen是1的情况，处理方式同上。
		if gen == bGen && idx < b.idx || gen+1 == bGen && idx >= b.idx || gen == maxGen && bGen == 1 && idx >= b.idx {
			chunkIdx := idx / chunkSize
			// 因为支持了缓存的持久化，所以带来了一定的复杂度
			// 贯彻简单至上，惜代码如今的v神为什么不惜引入这样的复杂度，还要支持这样不常用的功能
			// 从背景上来说，是为了支持VictoriaMetrics这个时序数据库
			// 从权衡上来说，v神在效率和复杂度的权衡中，这次选择了效率
			if chunkIdx >= uint64(len(b.chunks)) {
				// Corrupted data during the load from file. Just skip it.
				atomic.AddUint64(&b.corruptions, 1)
				goto end
			}
			chunk := b.chunks[chunkIdx]
			idx %= chunkSize
			if idx+4 >= chunkSize {
				// Corrupted data during the load from file. Just skip it.
				atomic.AddUint64(&b.corruptions, 1)
				goto end
			}
			kvLenBuf := chunk[idx : idx+4]
			keyLen := (uint64(kvLenBuf[0]) << 8) | uint64(kvLenBuf[1])
			valLen := (uint64(kvLenBuf[2]) << 8) | uint64(kvLenBuf[3])
			idx += 4
			if idx+keyLen+valLen >= chunkSize {
				// Corrupted data during the load from file. Just skip it.
				atomic.AddUint64(&b.corruptions, 1)
				goto end
			}
			if string(k) == string(chunk[idx:idx+keyLen]) {
				idx += keyLen
				if returnDst {
					dst = append(dst, chunk[idx:idx+valLen]...)
				}
				found = true
			} else {
				atomic.AddUint64(&b.collisions, 1)
			}
		}
	}
end:
	b.mu.RUnlock()
	if !found {
		atomic.AddUint64(&b.misses, 1)
	}
	return dst, found
}
```

因为底层使用ring buffer存储，过期的缓存会被覆盖掉，所以删除操作，只需要将其索引删除即可

```go
func (c *Cache) Del(k []byte) {
	h := xxhash.Sum64(k)
	idx := h % bucketsCount
	c.buckets[idx].Del(h)
}

func (b *bucket) Del(h uint64) {
	b.mu.Lock()
	delete(b.m, h)
	b.mu.Unlock()
}
```

清理索引的map

```go
// 每8KiB次bucket.set执行一次，用以删除过期的hash
func (b *bucket) Clean() {
	b.mu.Lock()
	bGen := b.gen & ((1 << genSizeBits) - 1)
	bIdx := b.idx
	bm := b.m
	for k, v := range bm {
		gen := v >> bucketSizeBits
		idx := v & ((1 << bucketSizeBits) - 1)
		// 跳过在ring buffer中的缓存项目
		if gen == bGen && idx < bIdx || gen+1 == bGen && idx >= bIdx || gen == maxGen && bGen == 1 && idx >= bIdx {
			continue
		}
		delete(bm, k)
	}
	b.mu.Unlock()
}
```

## SetBig & GetBig

```go
// SetBig将一个大块分为较小的64KiB子块subValues，
// 每个块保存这样的缓存项
// Key            -> Value
// (valueHash, 0) -> subValue1
// (valueHash, 1) -> subVlaue2
// ...
// (valueHash, N) -> subValueN
// 为了索引这些子块，存入下面这个缓存项
// keyHash -> (valueHash, valueLen)
// 这样Get就能通过keyHash拿到整个value
func (c *Cache) SetBig(k, v []byte) {
	atomic.AddUint64(&c.bigStats.SetBigCalls, 1)
	if len(k) > maxKeyLen {
		atomic.AddUint64(&c.bigStats.TooBigKeyErrors, 1)
		return
	}
	valueLen := len(v)
	valueHash := xxhash.Sum64(v)

	// Split v into chunks with up to 64Kb each.
	subkey := getSubkeyBuf()
	var i uint64
	for len(v) > 0 {
		// maxSubvalueLen 中的16
		subkey.B = marshalUint64(subkey.B[:0], valueHash)
		subkey.B = marshalUint64(subkey.B, uint64(i))
		i++
		subvalueLen := maxSubvalueLen
		if len(v) < subvalueLen {
			subvalueLen = len(v)
		}
		subvalue := v[:subvalueLen]
		v = v[subvalueLen:]
		c.Set(subkey.B, subvalue)
	}

	// Write metavalue, which consists of valueHash and valueLen.
	subkey.B = marshalUint64(subkey.B[:0], valueHash)
	subkey.B = marshalUint64(subkey.B, uint64(valueLen))
	c.Set(k, subkey.B)
	putSubkeyBuf(subkey)
}
```

## getChunk & putChunk

fastcache使用mmap分配内存并池化为chunk使用，避免GOGC的涉入。

```go
// 每次至少分配1024块。
const chunksPerAlloc = 1024

var (
	freeChunks     []*[chunkSize]byte
	freeChunksLock sync.Mutex
)

func getChunk() []byte {
	freeChunksLock.Lock()
	if len(freeChunks) == 0 {
		// Allocate offheap memory, so GOGC won't take into account cache size.
		// This should reduce free memory waste.
		data, err := syscall.Mmap(-1, 0, chunkSize*chunksPerAlloc, syscall.PROT_READ|syscall.PROT_WRITE, syscall.MAP_ANON|syscall.MAP_PRIVATE)
		if err != nil {
			panic(fmt.Errorf("cannot allocate %d bytes via mmap: %s", chunkSize*chunksPerAlloc, err))
		}
		for len(data) > 0 {
			p := (*[chunkSize]byte)(unsafe.Pointer(&data[0]))
			freeChunks = append(freeChunks, p)
			data = data[chunkSize:]
		}
	}
	n := len(freeChunks) - 1
	p := freeChunks[n]
	freeChunks[n] = nil
	freeChunks = freeChunks[:n]
	freeChunksLock.Unlock()
  // 虚心请教，有谁能告诉我为什么返回p[:]，而不是p
	return p[:]
}

func putChunk(chunk []byte) {
	if chunk == nil {
		return
	}
	chunk = chunk[:chunkSize]
	p := (*[chunkSize]byte)(unsafe.Pointer(&chunk[0]))

	freeChunksLock.Lock()
	freeChunks = append(freeChunks, p)
	freeChunksLock.Unlock()
}

```

## Save & Load

SaveToFile调用SaveToFileConcurrent实现，其以bucket为单位存储data.%d.bin文件，实现编码过程和I/O的并发。另一个值得学习的点是，fastcache没有直接去写filePath，而是先去写tmpDir，最后将tmpDir rename到filePath，这样能保证，如果写成功，用户就能看到内容，若失败，则什么都看不到（有种原子操作的感觉）。另外tmpDir常以tmpfs实现，tmpfs是在内存中的文件系统，这样rename时会进行一次完整I/O，而不会是小量多次的I/O。

```go
func (c *Cache) SaveToFileConcurrent(filePath string, concurrency int) error {
	// Create dir if it doesn't exist.
	dir := filepath.Dir(filePath)
	if _, err := os.Stat(dir); err != nil {
		if !os.IsNotExist(err) {
			return fmt.Errorf("cannot stat %q: %s", dir, err)
		}
		if err := os.MkdirAll(dir, 0755); err != nil {
			return fmt.Errorf("cannot create dir %q: %s", dir, err)
		}
	}

	// Save cache data into a temporary directory.
	tmpDir, err := ioutil.TempDir(dir, "fastcache.tmp.")
	if err != nil {
		return fmt.Errorf("cannot create temporary dir inside %q: %s", dir, err)
	}
	defer func() {
		if tmpDir != "" {
			_ = os.RemoveAll(tmpDir)
		}
	}()
	gomaxprocs := runtime.GOMAXPROCS(-1)
	if concurrency <= 0 || concurrency > gomaxprocs {
		concurrency = gomaxprocs
	}
	if err := c.save(tmpDir, concurrency); err != nil {
		return fmt.Errorf("cannot save cache data to temporary dir %q: %s", tmpDir, err)
	}

	// Remove old filePath contents, since os.Rename may return
	// error if filePath dir exists.
	if err := os.RemoveAll(filePath); err != nil {
		return fmt.Errorf("cannot remove old contents at %q: %s", filePath, err)
	}
	if err := os.Rename(tmpDir, filePath); err != nil {
		return fmt.Errorf("cannot move temporary dir %q to %q: %s", tmpDir, filePath, err)
	}
	tmpDir = ""
	return nil
}
```

## 总结

1. 简化设计，支持最通用的KV形式，并同时为用户复用内存提供接口；
2. 预先使用mmap分配内存，缓存使用期间没有gc消耗；
3. 缓存分成64KiB块，更适合现代CPU的cache做缓存；
4. 分桶降低锁粒度，提升效率；
5. 使用Ring Buffer，避免复杂淘汰机制带来的性能开销；
5. 持久化使用文件事务模式，先在临时目录构建，最后rename到目标目录，尽可能减少出现不一致的情况；

整个fastcache的代码不过千行，用的都是很平常的技术，但就是获得了很高的效率。简单至上(simple, but not trivial)。
