---
title: Golang map 三板斧第三式：实现原理
categories: [Golang]
tags: [Golang]
---

以 [Go 1.15](https://blog.golang.org/go1.15)

## 1.数据结构

## 1.1 简介

Go map 底层实现方式是 Hash 表（C++ map 基于红黑树实现，而 C++ 11 新增的 unordered\_map 则与 Go map 类似，都是基于 Hash 表实现）。Go map 的数据被置入一个由桶组成的有序数组中，每个桶最多可以存放 8 个 key/value 对。key 的 Hash 值低位用于在该数组中定位到桶，而高 8 位则用于在桶中区分 key/value 对。

Go map 的 hash 表中的基本单位是桶，每个桶最多存 8 个键值对，超了则会链接到额外的溢出桶。所以 Go map 基本数据结构是**hash数组+桶内的key-value数组+溢出的桶链表**。

当 Hash 表超过阈值需要扩容增长时，会分配一个新的桶数组，新数组的大小一般是旧数组的 2 倍。这里从旧数组将数据迁移到新数组，不会一次全量拷贝，因为耗时太大，Go 会在每次读写 Map 时以桶为单位做动态搬迁。

## 1.2 核心结构

map 主要有两个核心结构，基础结构和桶结构： hmap：map 的基础结构。 bmap：存放 key-value 的桶结构。严格来说 hmap.buckets 指向桶组成的数组，每个桶的头部是 bmap，之后是 8 个key，再是 8个 value，最后是 1 个溢出桶指针，指向额外的桶链表，用于存储溢出的元素。

map 的数据结构定义于 src/runtime/map.go 中，首先我们看下相关常量、hmap 和 bmap 的定义。

常量定义：

```golang
const (
	// Maximum number of key/elem pairs a bucket can hold.
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits

	// Maximum average load of a bucket that triggers growth is 6.5.
	// Represent as loadFactorNum/loadFactorDen, to allow integer math.
	loadFactorNum = 13
	loadFactorDen = 2

	// Possible tophash values. We reserve a few possibilities for special marks.
	// Each bucket (including its overflow buckets, if any) will have either all or none of its
	// entries in the evacuated* states (except during the evacuate() method, which only happens
	// during map writes and thus no one else can observe the map during that time).
	emptyRest      = 0 // this cell is empty, and there are no more non-empty cells at higher indexes or overflows.
	emptyOne       = 1 // this cell is empty
	evacuatedX     = 2 // key/elem is valid.  Entry has been evacuated to first half of larger table.
	evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
	evacuatedEmpty = 4 // cell is empty, bucket is evacuated.
	minTopHash     = 5 // minimum tophash for a normal filled cell.

	// flags
	iterator     = 1 // there may be an iterator using buckets
	oldIterator  = 2 // there may be an iterator using oldbuckets
	hashWriting  = 4 // a goroutine is writing to the map
	sameSizeGrow = 8 // the current map growth is to a new map of the same size

	// sentinel bucket ID for iterator checks
	noCheck = 1<<(8*sys.PtrSize) - 1
)
```

常量说明：

```golang
// map 的基本属性设定
bucketCnt：表示一个桶最多存储 8 个 key-value 对
loadFactorNum/loadFactorDen：表示装载因子为 6.5，即元素数量超过（桶数量*6.5） 时将触发 map 扩容。用两个整数表示装载因子，原因是可用于整数表达式

// 以下为元素 tophash 的可能取值，表示元素的状态
emptyRest：当前元素已被删除且在桶链表上的更高下标的元素均不可用，这个说明当前元素的位置可以被重新使用
emptyOne：当前元素已被删除但是位置不可被使用
evacuatedX：当前元素将搬迁到新hash数组的左半部分
evacuatedY：当前元素将搬迁到新hash数组的右半部分
evacuatedEmpty：当前元素已搬迁到新扩容的桶
minTopHash：元素 tophash 最小值

// 以下为 map.flags 的位标识，表示 map 的状态
iterator：可能有一个迭代器使用桶
oldIterator：可能有一个迭代器使用旧桶
hashWriting：一个 Go 程正在写 map
sameSizeGrow：正在向同大小的 map 做扩容
```

map 定义：

```golang
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}

// mapextra holds fields that are not present on all maps.
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}
```

hmap 说明：

```golang
count：元素的个数。len() 函数返回的就是这个值
flags：状态标记位。如是否被多线程读写、迭代器在使用新桶、迭代器在使用旧桶等
B：桶指数，表示 hash 数组中桶数量为 2^B（不包括溢出桶）。最大可存储元素数量为 loadFactor * 2^B
noverflow：溢出桶的数量的近似值。详见函数 incrnoverflow()
hash0：hash种子

buckets：指向2^B个桶组成的数组的指针。可能是 nil 如果 count 为 0
oldbuckets：指向长度为新桶数组一半的旧桶数组，仅在增长时为非零
nevacuate：进度计数器，表示扩容后搬迁的进度（小于该数值的桶已迁移）

extra.overflow：保存溢出桶链表
extra.oldoverflow：保存旧溢出桶链表
extra.nextOverflow：下一个空闲溢出桶地址
```

bmap 定义：

```golang
// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```

bmap 说明：

```golang
tohash：	存储桶内 8 个 key 的 hash 值的高字节。tophash[0] < minTopHash 表示桶处于扩容迁移状态
```

特别注意：实际分配内存时会申请一个更大的内存空间 A，A 的前 8 字节为 bmap，后面依次跟 8 个key、8 个 value、1 个溢出指针。把所有的 key 排在一起和所有的 value 排列在一起，而不是交替排列（key/elem/key/elem/…），这样可以填充空白字节，例如 map\[int64\]int8。map 的桶结构实际指的是内存空间 A。

另外，map.go 里很多函数的第 1 个入参是下面这个结构，从成员来看很明显，此结构标示了键值对和桶的类型和大小等必要信息。有了这个结构的信息，map.go 的代码就可以与键值对的具体数据类型解耦。所以 map.go 用内存偏移量和 unsafe.Pointer 指针来直接对内存进行存取，而无需关心 key 或 value 的具体类型。

```golang
type maptype struct {
	typ    _type
	key    *_type
	elem   *_type
	bucket *_type // internal type representing a hash bucket
	// function for hashing keys (ptr to key, seed) -> hash
	hasher     func(unsafe.Pointer, uintptr) uintptr
	keysize    uint8  // size of key slot
	elemsize   uint8  // size of elem slot
	bucketsize uint16 // size of bucket
	flags      uint32
}
```

C++ 使用模板可以根据不同的类型生成 map 的代码。Golang 则通过上述 maptype 结构体传递键值对的类型大小等信息，从而 map.go 直接用指针操作对应大小的内存来实现全局一份 map 代码同时适用于不同类型的键值对。相比于 C++ 用模板实现 map 的方式，Go map 的目标文件的代码量会更小。

## 1.3 数据结构图

创建 map 时，会初始化一个 hmap 结构体，同时分配一个足够大的内存空间 A。其中 A 的前段用于 hash 数组，A 的后段预留给溢出的桶。于是 `hmap.buckets` 指向 hash 数组，即 A 的首地址；`hmap.extra.nextOverflow` 初始时指向内存 A 中的后段，即 hash 数组结尾的下一个桶，也即第 1 个预留的溢出桶。所以当 hash 冲突需要使用到新的溢出桶时，会优先使用上述预留的溢出桶。`hmap.extra.nextOverflow` 依次往后偏移直到用完所有的溢出桶，才有可能会申请新的溢出桶空间。

![](/assets/img/posts/golang-map.png)

上图中，当需要分配一个溢出桶时，会优先从预留的溢出桶数组里取一个出来链接到链表后面，这时不需要再次申请内存。但当预留的溢出桶被用完了，则需要申请新的溢出桶。

## 2.实现机制

## 2.1 创建

使用`make(map[k]v)`或`make(map[k]v, hint)`且`hint<=8`创建 map 时调用`makemap_small()`。使用`make(map[k]v, hint)`且`hint>8`时，会调用`makemap()`函数。

值得注意的是，makemap() 创建的 hash 数组，数组的前面是 hash 表的空间，当 hint >= 4 时后面会追加 2^(hint-4) 个桶，之后进行内存页对齐又追加了若干个桶，所以创建 map 时一次内存分配既分配了用户预期大小的 hash 数组，又追加了一定量的预留的溢出桶，还做了内存对齐，一举多得。

```golang
// makemap_small implements Go map creation for make(map[k]v) and
// make(map[k]v, hint) when hint is known to be at most bucketCnt
// at compile time and the map needs to be allocated on the heap.
func makemap_small() *hmap {
	h := new(hmap)
	h.hash0 = fastrand()
	return h
}

// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
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

// makeBucketArray initializes a backing array for map buckets.
// 1<<b is the minimum number of buckets to allocate.
// dirtyalloc should either be nil or a bucket array previously
// allocated by makeBucketArray with the same t and b parameters.
// If dirtyalloc is nil a new backing array will be alloced and
// otherwise dirtyalloc will be cleared and reused as backing array.
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	// For small b, overflow buckets are unlikely.
	// Avoid the overhead of the calculation.
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
		// dirtyalloc was previously generated by
		// the above newarray(t.bucket, int(nbuckets))
		// but may not be empty.
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		if t.bucket.ptrdata != 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

	if base != nbuckets {
		// We preallocated some overflow buckets.
		// To keep the overhead of tracking these overflow buckets to a minimum,
		// we use the convention that if a preallocated overflow bucket's overflow
		// pointer is nil, then there are more available by bumping the pointer.
		// We need a safe non-nil pointer for the last overflow bucket; just use buckets.
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

## 2.2 增加或修改

go map 的插入操作，调用`mapassign()`函数。

go map 的插入或修改我们需要注意两点： （1）hmap 指针传递的方式，决定了 map 在使用前必须初始化，对空 map 插入会 panic； （2）go map 不支持并发读写，会导致不可恢复的异常（终止程序）。如果一定要并发，请用 sync.Map 或自己解决冲突。

上述两个限制，在`mapassign()`函数开头能找到答案。阅读 `mapassign()`函数，可以看出 map 的插入或修改经过如下几个步骤：

（1）参数合法性检测与 hash 值计算。 如果 map 为 nil 或存在并发读写都将引发 panic。如果参数合法，则计算 key 的 hash 值来确定 key 的具体位置。然后置 hashWriting 标志，key 写入 buckets 后才会清除标志。map 不能为空，但 hash 数组初始值可以是空的，mapassign() 函数如果检测到 hash 数组为空，则进行初始化。

```golang
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(mapassign)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled {
		msanread(key, t.key.size)
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	hash := t.hasher(key, uintptr(h.hash0))

	// Set hashWriting after calling t.hasher, since t.hasher may panic,
	// in which case we have not actually done a write.
	h.flags ^= hashWriting

	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}
	...
}
```

（2）定位 key 在 hash 表中的位置。 用 key 的 hash 值的低位定位 hash 数组的下标偏移量，用 hash 值的高 8 位用于在桶内定位键值对。

```golang
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	...
again:
	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
	top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var elem unsafe.Pointer
	...
}
```

（3）进一步定位 key 可以插入的桶及桶中的位置。 两轮循环，外层循环遍历 hash 桶及其指向的溢出链表，内层循环则在桶内遍历（一个桶最多 8 个 key-value 对）。

如果在链表上的桶内找到了 key，则直接更新 key。如果没有找到 key 需要新增的话，那么会进入第 4 步插入新 key。

```golang
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	...
bucketloop:
	for {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				}
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if !t.key.equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			goto done
		}
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}
	...
}
```

（4）插入新 key。 插入新 key 首先需要判断 map 是否需要扩容（扩容机制下文再详述）。如果不需要扩容且找到了 key 的插入位置，则直接插入，如果没有找到插入位置说明链表上的桶都满了，这时 inserti 为 nil，链接一个新的溢出桶进来。

注意：当 key 或 value 的大小超过一定值时，桶只存储 key 或 value 的指针。

```golang
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	...
	// Did not find mapping for key. Allocate new cell & add entry.

	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}

	if inserti == nil {
		// all current buckets are full, allocate a new one.
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		elem = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// store new key/elem at insert position
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectelem() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(elem) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++
	...
}
```

（5）结束插入。 先判断写标识是否还在，如果不在了表示存在并发写的情况，直接抛出异常，终止程序。然后释放 hashWriting 标志位，返回value可插入位置的指针。注意：value 还没插入。

```golang
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	...
done:
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.indirectelem() {
		elem = *((*unsafe.Pointer)(elem))
	}
	return elem
}
```

mapassign() 只插入 tophash 和 key，并返回 value 位置指针，编译器会在调用 mapassign() 后用汇编往 value 地址插入 value。

## 2.3 删除

删除与插入类似，前面的步骤都是参数和状态判断、定位 key/value 位置，然后 clear 对应的内存。不详细展开。以下是几个关键点： （1）删除过程中也会置 hashWriting 标志； （2）当 key/value 过大时，hash 表里存储的是指针，这时候用软删除，置指针为 nil，数据交给 GC 去删。当然，这是 map 的内部处理，外层是无感知的，外层已经拿到的数据不受影响，还能继续使用。

由于定位 key 位置的方式是查找 tophash，所以删除操作对 tophash 的处理是关键： （1）map 首先将对应位置的 tophash\[i\] 置为 emptyOne，表示该位置已被删除； （2）如果 tophash\[i\] 不是整个链表的最后一个有效位置，则只置 emptyOne 标志，该位置被删除但未释放，后续插入操作不能使用此位置；如果 tophash\[i\] 是链表最后一个有效位置了，则把链表最后面的所有标志为 emptyOne 的位置，都置为 emptyRest。置为 emptyRest 的位置可以在后续的插入操作中被使用。

这种删除方式，以少量空间避免了被删除的数据再次插入时出现数据移动的情况。事实上，Go 数据一旦被插入到桶的确切位置，map 是不会再移动该数据在桶中的位置了。

```golang
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	...
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if t.indirectkey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			if !t.key.equal(key, k2) {
				continue
			}
			// Only clear key if there are pointers in it.
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.key.ptrdata != 0 {
				memclrHasPointers(k, t.key.size)
			}
			e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
			if t.indirectelem() {
				*(*unsafe.Pointer)(e) = nil
			} else if t.elem.ptrdata != 0 {
				memclrHasPointers(e, t.elem.size)
			} else {
				memclrNoHeapPointers(e, t.elem.size)
			}
			b.tophash[i] = emptyOne
			// If the bucket now ends in a bunch of emptyOne states,
			// change those to emptyRest states.
			// It would be nice to make this a separate function, but
			// for loops are not currently inlineable.
			if i == bucketCnt-1 {
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}
			for {
				b.tophash[i] = emptyRest
				if i == 0 {
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// Find previous bucket, continue at its last entry.
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
			h.count--
			break search
		}
	}

	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}
```

## 2.4 查找

查找操作由 mapaccess 开头的一组函数实现。前文在插入和删除之前都得先定位查找到元素，逻辑是类似的，也比较简单，就不细说了：

```golang
mapaccess1()：通过 key 查找，返回 value 指针，用于 val := map[key]。未找到时返回value类型的零值
mapaccess2()：通过 key 查找，返回 value 指针，以及 bool 类型的是否查找成功的标志，用于 val, ok := map[key]。未找到时返回 value 类型的零值
mapaccessK()：通过 key 查找，返回 key 和 value 指针，用于迭代器(range)。未找到时返回空指针
mapaccess1_fat()，对 mapaccess1() 的封装，区别是 mapaccess1_fat() 多了个zero参数，未找到时返回 zero
mapaccess2_fat()，对 mapaccess1() 的封装。相比 mapaccess1_fat()，本函数增加一个是否查找成功的标志
```

## 2.5 迭代

map 的迭代是通过 hiter 结构和对应的两个辅助函数（`mapiterinit()` 和 `mapiternext()`）实现的。hiter 结构由编译器在调用辅助函数之前创建并传入，每次迭代结果也由 hiter 结构传回。下方 it 即 hiter 结构体的指针变量。

### 2.5.1 hiter

```golang
// A hash iteration structure.
// If you modify hiter, also change cmd/compile/internal/gc/reflect.go to indicate
// the layout of this structure.
type hiter struct {
	key         unsafe.Pointer // Must be in first position.  Write nil to indicate iteration end (see cmd/internal/gc/range.go).
	elem        unsafe.Pointer // Must be in second position (see cmd/internal/gc/range.go).
	t           *maptype
	h           *hmap
	buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
	bptr        *bmap          // current bucket
	overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
	oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
	startBucket uintptr        // bucket iteration started at
	offset      uint8          // intra-bucket offset to start from during iteration (should be big enough to hold bucketCnt-1)
	wrapped     bool           // already wrapped around from end of bucket array to beginning
	B           uint8
	i           uint8
	bucket      uintptr
	checkBucket uintptr
}
```

关键字段说明如下：

```golang
it.key：每次迭代的结果 key
it.elem：每次迭代的结果 value
it.t: map 的类型信息
it.h：map 地址
it.buckets：迭代初始化时的 hash 数组地址
it.h：map 地址
it.bptr：当前遍历的桶
it.overflow：当前 hash 数组的溢出桶
it.oldoverflow：旧 hash 数组的溢出桶
it.startBucket：开始桶的偏移量，表示遍历从这个 hash 桶开始
it.offset：桶内偏移量，表示每个桶的遍历都从这个偏移量开始
it.wrapped：hash 数组尾部已经遍历完开始折返从头遍历
it.B：迭代初始化时的 h.B
it.i：当前桶已遍历的键值对数量，i 初始为0，当 i=8 时表示这个桶遍历完了，将 it.bptr 移向下一个桶
it.bucket：当前 hash 桶偏移量
it.checkBucket：不为 noCheck(1<<(8*sys.PtrSize) - 1)的话，表示当前桶还没有搬迁到新 map，需要对旧桶检查跳过迁移到新扩容的hash桶的元素
```

### 2.5.2 mapiterinit()

`mapiterinit()`函数主要是决定我们从哪个位置开始迭代，为什么是从哪个位置，而不是直接从 hash 数组头部开始呢？hash 表中数据每次插入的位置是变化的，这是因为实现的原因，一方面 hash 种子是随机的，这导致相同的数据在不同的 map 变量内的 hash 值不同；另一方面即使同一个 map 变量内，数据删除再添加的位置也有可能变化，因为在同一个桶及溢出链表中数据的位置不分先后，所以为了防止用户错误的依赖于每次迭代的顺序，map 作者干脆让相同的 map 每次迭代的顺序也是随机的。

```golang
// mapiterinit initializes the hiter struct used for ranging over maps.
// The hiter struct pointed to by 'it' is allocated on the stack
// by the compilers order pass or on the heap by reflect_mapiterinit.
// Both need to have zeroed hiter since the struct contains pointers.
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiterinit))
	}

	if h == nil || h.count == 0 {
		return
	}

	if unsafe.Sizeof(hiter{})/sys.PtrSize != 12 {
		throw("hash_iter size incorrect") // see cmd/compile/internal/gc/reflect.go
	}
	it.t = t
	it.h = h

	// grab snapshot of bucket state
	it.B = h.B
	it.buckets = h.buckets
	if t.bucket.ptrdata == 0 {
		// Allocate the current slice and remember pointers to both current and old.
		// This preserves all relevant overflow buckets alive even if
		// the table grows and/or overflow buckets are added to the table
		// while we are iterating.
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	// decide where to start
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))

	// iterator state
	it.bucket = it.startBucket

	// Remember we have an iterator.
	// Can run concurrently with another mapiterinit().
	if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
		atomic.Or8(&h.flags, iterator|oldIterator)
	}

	mapiternext(it)
}
```

### 2.5.3 mapiternext()

map 的遍历由函数`mapiternext()`完成，过程如下： （1）从 hash 数组中第 it.startBucket 个桶开始，先遍历 hash 桶，然后是这个桶的溢出链表； （2）之后 hash 数组偏移量+1，继续前一步动作； （3）遍历每一个桶，无论是 hash 桶还是溢出桶，都从 it.offset 偏移量开始； （4）当迭代器经过一轮循环回到 it.startBucket 的位置，结束遍历。

注意： （1）map 如果在遍历开始时发现处于写入状态，那么报并发读写异常，终止程序。 （2）迭代还需要关注扩容的情况：如果是在迭代开始后才 growing，那么迭代初始状态如 it.buckets 和 it.B 等将被改变，迭代有可能出现异常。如果是先 growing，再开始迭代。这种情况下，不会出现异常，会先到旧 hash 表中检查 key 对应的桶有没有被迁移，未迁移则遍历旧桶，已迁移则遍历新 hash 表里对应的桶。

```golang
func mapiternext(it *hiter) {
	h := it.h
	if raceenabled {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiternext))
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map iteration and map write")
	}
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	checkBucket := it.checkBucket

next:
	if b == nil {
		if bucket == it.startBucket && it.wrapped {
			// end of iteration
			it.key = nil
			it.elem = nil
			return
		}
		if h.growing() && it.B == h.B {
			// Iterator was started in the middle of a grow, and the grow isn't done yet.
			// If the bucket we're looking at hasn't been filled in yet (i.e. the old
			// bucket hasn't been evacuated) then we need to iterate through the old
			// bucket and only return the ones that will be migrated to this bucket.
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
			if !evacuated(b) {
				checkBucket = bucket
			} else {
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
				checkBucket = noCheck
			}
		} else {
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
			checkBucket = noCheck
		}
		bucket++
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
	for ; i < bucketCnt; i++ {
		offi := (i + it.offset) & (bucketCnt - 1)
		if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {
			// TODO: emptyRest is hard to use here, as we start iterating
			// in the middle of a bucket. It's feasible, just tricky.
			continue
		}
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		if t.indirectkey() {
			k = *((*unsafe.Pointer)(k))
		}
		e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.elemsize))
		if checkBucket != noCheck && !h.sameSizeGrow() {
			// Special case: iterator was started during a grow to a larger size
			// and the grow is not done yet. We're working on a bucket whose
			// oldbucket has not been evacuated yet. Or at least, it wasn't
			// evacuated when we started the bucket. So we're iterating
			// through the oldbucket, skipping any keys that will go
			// to the other new bucket (each oldbucket expands to two
			// buckets during a grow).
			if t.reflexivekey() || t.key.equal(k, k) {
				// If the item in the oldbucket is not destined for
				// the current new bucket in the iteration, skip it.
				hash := t.hasher(k, uintptr(h.hash0))
				if hash&bucketMask(it.B) != checkBucket {
					continue
				}
			} else {
				// Hash isn't repeatable if k != k (NaNs).  We need a
				// repeatable and randomish choice of which direction
				// to send NaNs during evacuation. We'll use the low
				// bit of tophash to decide which way NaNs go.
				// NOTE: this case is why we need two evacuate tophash
				// values, evacuatedX and evacuatedY, that differ in
				// their low bit.
				if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
					continue
				}
			}
		}
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey() || t.key.equal(k, k)) {
			// This is the golden data, we can return it.
			// OR
			// key!=key, so the entry can't be deleted or updated, so we can just return it.
			// That's lucky for us because when key!=key we can't look it up successfully.
			it.key = k
			if t.indirectelem() {
				e = *((*unsafe.Pointer)(e))
			}
			it.elem = e
		} else {
			// The hash table has grown since the iterator was started.
			// The golden data for this key is now somewhere else.
			// Check the current hash table for the data.
			// This code handles the case where the key
			// has been deleted, updated, or deleted and reinserted.
			// NOTE: we need to regrab the key as it has potentially been
			// updated to an equal() but not identical key (e.g. +0.0 vs -0.0).
			rk, re := mapaccessK(t, h, k)
			if rk == nil {
				continue // key has been deleted
			}
			it.key = rk
			it.elem = re
		}
		it.bucket = bucket
		if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
			it.bptr = b
		}
		it.i = i + 1
		it.checkBucket = checkBucket
		return
	}
	b = b.overflow(t)
	i = 0
	goto next
}
```

## 2.6 扩容

Go map 的扩容缩容都是 grow 相关的函数来完成的。只有当新增 key 时，才有可能触发扩容。因为只有新增 key 时，才有可能触达最大负载系数或者有太多的溢出桶。

```golang
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	...
	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}
	...
}

// overLoadFactor reports whether count items placed in 1<<B buckets is over loadFactor.
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}

// tooManyOverflowBuckets reports whether noverflow buckets is too many for a map with 1<<B buckets.
// Note that most of these overflow buckets must be in sparse use;
// if use was dense, then we'd have already triggered regular map growth.
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	// If the threshold is too low, we do extraneous work.
	// If the threshold is too high, maps that grow and shrink can hold on to lots of unused memory.
	// "too many" means (approximately) as many overflow buckets as regular buckets.
	// See incrnoverflow for more details.
	if B > 15 {
		B = 15
	}
	// The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
	return noverflow >= uint16(1)<<(B&15)
}
```

从赋值函数`mapassign()`可以看出，触发扩容有两个条件： （1）当前不处在 growing 状态； （2.1）元素个数 count 大于 hash 桶数量(2^B)\*6.5。注意这里的 hash 桶指的是 hash 数组中的桶，不包括溢出的桶； （2.2）或溢出的桶数量 noverflow>=32768(1<<15) 或者 noverflow>=hash 数组中桶数量。

go map 的扩容预处理由函数 hashGrow() 来完成，主要完成两个操作： （1）判断扩容类型； （2）申请新的 hash 桶。 新申请的 hash 桶数组指针由 h.buckets 保存，h.oldbuckets 则指向旧 hash 桶数组。map 是否处于扩容状态是根据 h.oldbuckets 是否为空来判断的。

Go map 有两种扩容类型： （1）一种是真扩容，扩到 hash 桶数量为原来的两倍，针对元素数量过多的情况； （2）一种是假扩容，hash 桶数量不变，只是把元素搬迁到新的 map，针对溢出桶过多的情况。如果是假扩容，那么 hmap.flags 会被打上 sameSizeGrow 标识。

```golang
func hashGrow(t *maptype, h *hmap) {
	// If we've hit the load factor, get bigger.
	// Otherwise, there are too many overflow buckets,
	// so keep the same number of buckets and "grow" laterally.
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
	// commit the grow (atomic wrt gc)
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
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

	// the actual copying of the hash table data is done incrementally
	// by growWork() and evacuate().
}
```

扩容的预处理工作已经完成，接下来就是怎么把元素搬迁到新 hash 表里了。如果现在就一次全量搬迁过去，显然接下来会有比较长的一段时间 map 被占用（不支持并发）。所以搬迁的工作是增量搬迁的，这个从 hashGrow() 函数尾部的注释也可以看出。

在插入和删除的函数内都有下面一段代码用于在每次插入和删除操作时，执行一次搬迁工作：

```golang
if h.growing() {
	growWork(t, h, bucket)
}

func growWork(t *maptype, h *hmap, bucket uintptr) {
	// make sure we evacuate the oldbucket corresponding
	// to the bucket we're about to use
	evacuate(t, h, bucket&h.oldbucketmask())

	// evacuate one more oldbucket to make progress on growing
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}
```

（1）每执行一次插入或删除，都会调用 growWork() 函数搬迁 0~2 个 hash 桶（有可能这次需要搬迁的 2 个桶在此之前都被搬过了）； （2）搬迁是以 hash 桶为单位的，包含对应的 hash 桶和这个桶的溢出链表； （3）被 delete 掉的元素（emptyone 标志）会被舍弃不进行搬迁。

## 3.小结

## 3.1 基本原理

Go map 的实现原理有如下要点： （1）底层数据结构为**hash数组 + 桶 + 溢出的桶链表**，每个桶存储最多8个key-value对； （2）查找和插入的原理：key 的 hash 值（低阶位）与桶数量相与，得到 key 所在的 hash 桶，再用key 的高 8 位与桶中的 tophash\[i\] 对比，相同则进一步对比key值，key 值相等则找到； （3）delete 操作只置删除标志位（emptyOne）且不能被使用，防止被删除的元素再次插入时出现移动； （4）Go map 不支持并发。插入（更新）、删除、搬迁等操作会置 hashWriting 标志，检测到并发直接 panic； （5）每次扩容 hash 表增大 1 倍，hash 表只增不减； （6）扩容类型有两种：一种是真扩容，扩到 hash 桶数量为原来的两倍，针对元素数量过多的情况；一种是假扩容，hash 桶数量不变，只是把元素搬迁到新的 map，针对溢出桶过多的情况。

## 3.2 时间与空间复杂度

（1）时间复杂度。 go map 是 hash 实现，我们先不管具体原理，江湖人人皆知基于 hash 实现的算法其时间复杂度均为 O(1)。

正常情况，且不考虑扩容状态，复杂度O(1)：通过hash值定位桶是O(1)，一个桶最多8个元素，合理的hash算法应该能把元素相对均匀散列，所以溢出链表（如果有）也不会太长，所以虽然在桶和溢出链表上定位key是遍历，考虑到数量小也可以认为是O(1)。

正常情况，处于扩容状态时，复杂度也是O(1)：相比于上一种状态，扩容会增加搬迁最多 2 个桶和溢出链表的时间消耗，当溢出链表不太长时，复杂度也可以认为是 O(1)。

极端情况，散列极不均匀，大部分数据被集中在一条散列链表上，复杂度退化为O(n)。

Go 采用的 hash 算法是很成熟的算法，极端情况暂不考虑。所以综合情况下 Go map 的时间复杂度为 O(1)。

（2）空间复杂度。 首先我们不考虑因删除大量元素导致的空间浪费情况，因为删除只是值 key 的标志为 emptyOne，这种情况现在 Go 是留给程序员自己解决，所以这里只考虑一个持续增长状态的 map 的一个空间使用率： 由于溢出桶数量超过 hash 桶数量时会触发假扩容，所以最坏的情况是数据被集中在一条链上，hash表基本是空的，这时空间浪费 O(n)。

最好的情况下，数据均匀散列在 hash 表上，没有元素溢出，这时最好的空间复杂度就是负载因子决定了，当前 Go 的负载因子由全局变量决定，即 loadFactorNum/loadFactorDen = 6.5。即平均每个hash 桶被分配到 6.5 个元素以上时，开始扩容。所以最小的空间浪费是(8-6.5)/8 = 0.1875，即O(0.1875n)

**结论：** Go map 的空间复杂度（指除去正常存储元素所需空间之外的空间浪费）是 O(0.1875n) ~ O(n) 之间。

