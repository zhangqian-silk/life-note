# 哈希表

哈希表是一系列键值对的无序集合，维护了 key 和 value 的映射关系，其中 key 不能重复，还提供了常数时间复杂度的读写性能。

## 数据结构

### hamp

在 Golang 中，`map` 类型实质上就是哈希表。数据结构 [hamp](https://github.com/golang/go/blob/e3d87d19320001e6081449550292d76ef660ab03/src/runtime/map.go#L109) 如下所示：

- `count`：当前哈希表的大小，即哈希表中元素的数量，可以通过 `len` 函数获取
- `flags`：当前哈希表的状态，用于并发控制
- `B`：当前桶的数量，其中 $2^B=len(buckets)$
- `noverflow`：溢出桶的近似数量，当溢出桶数量较多时，会触发扩容操作
- `hash0`：哈希种子，增加随机性
- `buckets`：指向桶数组，数组元素类型为 `bmap`
- `oldBuckets`：扩容时，指向扩容前的桶数组
- `nevacuate`：扩容时的数据迁移的进度，小于该数值的桶已经完成了迁移操作
- `extra`：可选字段，用于保存溢出桶的地址，防止溢出桶被 GC 回收

```go
const (
    // flags
    iterator     = 1 // there may be an iterator using buckets
    oldIterator  = 2 // there may be an iterator using oldbuckets
    hashWriting  = 4 // a goroutine is writing to the map
    sameSizeGrow = 8 // the current map growth is to a new map of the same size
)

// A header for a Go map.
type hmap struct {
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
```

### bmap

哈希函数会将不同的输入值，映射到一个定长的输出值，输入值不同时，哈希值有可能相同，被称作哈希碰撞，但是当哈希值不同时，输入值一定不同，通过对比哈希值，则可以加快数据匹配效率。

Golang 在最终查询哈希表中数据时，会先通过低 B 位确认桶号，再通过高 8 位来进行初步数据匹配，以提高查询效率。

相对应的，桶的数据结构为 [bmap](https://github.com/golang/go/blob/377646589d5fb0224014683e0d1f1db35e60c3ac/src/runtime/map.go#L143)，其中 `tophash` 则用来存储哈希值的高 8 位的数组，加速数据匹配，数组长度为 `abi.MapBucketCount`，即 8 位，表示一个桶最多存放 8 个元素。

```go
const (
    // Maximum number of key/elem pairs a bucket can hold.
    MapBucketCountBits = 3 // log2 of number of elements in a bucket.
    MapBucketCount     = 1 << MapBucketCountBits
)

type bmap struct {
    tophash [abi.MapBucketCount]uint8
}
```

在某些特殊情况下，桶中的 `tophash` 还会被用来存储特定的[标记位](https://github.com/golang/go/blob/377646589d5fb0224014683e0d1f1db35e60c3ac/src/runtime/map.go#L86)：

- `emptyRest`：当前位置以及后续位置，包括溢出桶，都是空值
- `emptyOne`：当前位置是空值
- `evacuatedX` & `evacuatedY`：数据已经发生了迁移，位于新桶的前半部分或后半部分
- `evacuatedEmpty`：当前位置是空值，且桶已经进行了数据迁移
- `minTopHash`：高位哈希值的最小值，如果最终计算出的哈希值小于该数值，则会加上 `minTopHash` 用作新的高位哈希值，用来和其他标记位做区分

```go
const (
    // Possible tophash values. We reserve a few possibilities for special marks.
    emptyRest      = 0 // this cell is empty, and there are no more non-empty cells at higher indexes or overflows.
    emptyOne       = 1 // this cell is empty
    evacuatedX     = 2 // key/elem is valid.  Entry has been evacuated to first half of larger table.
    evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
    evacuatedEmpty = 4 // cell is empty, bucket is evacuated.
    minTopHash     = 5 // minimum tophash for a normal filled cell.

)
```

此外，因为哈希表中存储的元素的数据结构不固定，所以桶具体的数据结构在编译期才会动态确认，类型生成方法为 [MapBucketType](https://github.com/golang/go/blob/377646589d5fb0224014683e0d1f1db35e60c3ac/src/cmd/compile/internal/reflectdata/reflect.go#L73)，最终的数据结构如下所示：

- `keys` & `elems`：最终存储的键值对的数组，数组内元素类型为 key 和 value 对应的类型，数组长度同样为 8 位，与 `tophash` 保持一致
- `overflow`：指向溢出桶，当发生哈希碰撞，且当前桶中元素已经超过 8 个时，会临时建立溢出桶，通过链表的方式进行维护

```go
func MapBucketType(t *types.Type) *types.Type {
    // Builds a type representing a Bucket structure for
    // the given map type. This type is not visible to users -
    // we include only enough information to generate a correct GC
    // program for it.
    // Make sure this stays in sync with runtime/map.go.

    //     A "bucket" is a "struct" {
    //             tophash [abi.MapBucketCount]uint8
    //             keys [abi.MapBucketCount]keyType
    //             elems [abi.MapBucketCount]elemType
    //             overflow *bucket
    //         }
    ...
}
```

### mapextra

在 [MapBucketType](https://github.com/golang/go/blob/377646589d5fb0224014683e0d1f1db35e60c3ac/src/cmd/compile/internal/reflectdata/reflect.go#L73) 中，对于 `overflow` 的类型生成有如下一段逻辑

```go
func MapBucketType(t *types.Type) *types.Type {
    ...
    // If keys and elems have no pointers, the map implementation
    // can keep a list of overflow pointers on the side so that
    // buckets can be marked as having no pointers.
    // Arrange for the bucket to have no pointers by changing
    // the type of the overflow field to uintptr in this case.
    // See comment on hmap.overflow in runtime/map.go.
    otyp := types.Types[types.TUNSAFEPTR]
    if !elemtype.HasPointers() && !keytype.HasPointers() {
        otyp = types.Types[types.TUINTPTR]
    }
    overflow := makefield("overflow", otyp)
    ...
}
```

即对于 `bmap` 中的 `overflow` 字段，会自动根据哈希表中的元素类型，来生成相对应的指针类型，当 key 或者 value 包含指针时，`overflow` 为 `unsafe.Pointer` 类型，直接指向溢出桶。

而当 key 和 value 均不包含指针时，`overflow` 为 `uintptr` 类型，此时整个 `bmap` 不存在任何指针元素（对于 GC 来说，`uintptr` 不会被认为是引用类型），从而避免了 GC 时的扫描成本。

但是对于溢出桶来说，此时会存在 GC 的问题，故需要另外一个存储结构，直接引用这些溢出桶，即 `hamp` 中的一个可选的 `extra` 字段，数据类型为 [mapextra](https://github.com/golang/go/blob/377646589d5fb0224014683e0d1f1db35e60c3ac/src/runtime/map.go#L126)：

```go
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

当 `bmap` 中不包含指针时，即 key 和 elem 均不包含指针，overflow 类型为 `uintptr`，`hmap.extra` 中的 `overflow` 和 `oldoverflow` 字段，会分别引用 `hmap.buckets` 和 `hmap.oldbuckets` 中的溢出桶。

`nextOverFlow` 字段，则用于在内存预分配时，引用溢出桶数组中首个可用的溢出桶，并在之后使用的过程中，保持动态更新

## 初始化

Golang 支持通过字面量或者 `make` 函数来初始化哈希表：

```go
m1 := map[string]int{
    "key_m1_1": 1,
    "key_m1_2": 2,
}

m2 := make(map[string]int)
m2["key_m2_1"] = 1
m2["key_m2_2"] = 2

m3 := make(map[string]int, 4)
```

当未指定元素数量，或元素数量小于一个桶中元素的数量（8 个）时，会调用 [makemap_small](https://github.com/golang/go/blob/377646589d5fb0224014683e0d1f1db35e60c3ac/src/runtime/map.go#L296C6-L296C19) 来创建哈希表，此时仅会创建一个空的哈希表，并指定哈希种子，在编译时才会具体分配其他属性的内存空间。

```go
// makemap_small implements Go map creation for make(map[k]v) and
// make(map[k]v, hint) when hint is known to be at most bucketCnt
// at compile time and the map needs to be allocated on the heap.
func makemap_small() *hmap {
    h := new(hmap)
    h.hash0 = uint32(rand())
    return h
}
```

对于其他场景，最终会调用 [makemap](https://github.com/golang/go/blob/377646589d5fb0224014683e0d1f1db35e60c3ac/src/runtime/map.go#L318) 函数来创建哈希表：

- 先进行内存空间溢出判断，此时按照极限情况下，每个桶中仅分配一个元素来计算，如果有溢出风险，则将 `hint` 值置为 0，按照最小值来分配内存
- 初始化哈希表，并指定随机种子，与 `makemap_small` 逻辑一致
- 根据 `hint` 值，计算桶的个数，如果计算出的 B 大于 0，则预分配桶数组对应的内存空间

```go
// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
    mem, overflow := math.MulUintptr(uintptr(hint), t.Bucket.Size_)
    if overflow || mem > maxAlloc {
        hint = 0
    }

    if h == nil {
        h = new(hmap)
    }
    h.hash0 = uint32(rand())

    B := uint8(0)
    for overLoadFactor(hint, B) {
        B++
    }
    h.B = B

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

### 负载因子

在计算桶的数量时，会综合考虑内存的利用率与查找时的效率，桶的数量过多，显然会存在内存浪费的情况，而桶的数量过少，当出现碰撞时，则会增加溢出桶的数量，增加每次查找时所需比较的元素数量，进而减少查找效率。

对于哈希表中桶与元素的数量关系，有个核心指标叫做负载因子（loadFactor），用元素个数除以哈希表的容量来表示，即$loadFactor = num_{elems} / num_{buckets}$。

对于 Golang 来说，负载因子又可以表示为 $loadFactor = num_{elems} / (2^B)$，此外，每个桶中最多可以容纳 8 个元素，负载因子的最大值也同样为 8。

对于负载因子具体的取值，可以参考注释中给出的[测试报告](https://github.com/golang/go/blob/377646589d5fb0224014683e0d1f1db35e60c3ac/src/runtime/map.go#L33)：

- loadFactor：负载因子
- %overflow：溢出率
- bytes/entry：平均每对键值对消耗的字节数
- hitprobe：查找一个存在的 key 时，平均查找个数
- missprobe：查找一个不存在的 key 时，平均查找个数

```go
// Picking loadFactor: too large and we have lots of overflow
// buckets, too small and we waste a lot of space. I wrote
// a simple program to check some stats for different loads:
// (64-bit, 8 byte keys and elems)
//  loadFactor    %overflow  bytes/entry     hitprobe    missprobe
//        4.00         2.13        20.77         3.00         4.00
//        4.50         4.05        17.30         3.25         4.50
//        5.00         6.85        14.77         3.50         5.00
//        5.50        10.55        12.94         3.75         5.50
//        6.00        15.27        11.67         4.00         6.00
//        6.50        20.90        10.79         4.25         6.50
//        7.00        27.14        10.15         4.50         7.00
//        7.50        34.03         9.73         4.75         7.50
//        8.00        41.10         9.40         5.00         8.00
//
// %overflow   = percentage of buckets which have an overflow bucket
// bytes/entry = overhead bytes used per key/elem pair
// hitprobe    = # of entries to check when looking up a present key
// missprobe   = # of entries to check when looking up an absent key

const (
    // Maximum average load of a bucket that triggers growth is bucketCnt*13/16 (about 80% full)
    // Because of minimum alignment rules, bucketCnt is known to be at least 8.
    // Represent as loadFactorNum/loadFactorDen, to allow integer math.
    loadFactorDen = 2
    loadFactorNum = loadFactorDen * abi.MapBucketCount * 13 / 16
)
```

最终综合考虑各项指标，最终选择了 6.5 作为负载因子的默认值，并且为了方便进行整数运算，优化了相关常量的表达式。

### 内存分配

在初始化时，内存的预分配，也会根据负载因子来进行判断，在 [overLoadFactor](https://github.com/golang/go/blob/377646589d5fb0224014683e0d1f1db35e60c3ac/src/runtime/map.go#L1179) 函数中，当元素数量小于一个桶时，直接返回 `false`，此时 B 的值为 0，相对应的桶的数量为 $2^0=1$，足够容纳所有元素。

当元素数量大于一个桶时，是否超过负载因子的判断从 $(num_{elems} / (2^B)) > loadFactor$ 优化为了判断 $num_{elems} > (loadFactorNum * (2^B / loadFactorDen))$，会扩大 B 的值，直至满足负载因子。

```go
func makemap(t *maptype, hint int, h *hmap) *hmap {
    ...
    B := uint8(0)
    for overLoadFactor(hint, B) {
        B++
    }
    h.B = B
    ...
}

// overLoadFactor reports whether count items placed in 1<<B buckets is over loadFactor.
func overLoadFactor(count int, B uint8) bool {
    return count > abi.MapBucketCount && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}

// bucketShift returns 1<<b, optimized for code generation.
func bucketShift(b uint8) uintptr {
    return uintptr(1) << (b & (goarch.PtrSize*8 - 1))
}
```

在计算出合适的桶的数量后，将通过 [makeBucketArray](https://github.com/golang/go/blob/377646589d5fb0224014683e0d1f1db35e60c3ac/src/runtime/map.go#L359) 函数来创建桶数组：

```go
func makemap(t *maptype, hint int, h *hmap) *hmap {
    ...
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

- 首先判断 B 的大小，如果 B 小于 4，说明整体数据量较少，溢出的概率较低，会省略溢出桶的分配，否则，将额外预创建 $2^{B-4}$ 个溢出桶

    ```go
    func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
        base := bucketShift(b)
        nbuckets := base
        if b >= 4 {
            nbuckets += bucketShift(b - 4)
            sz := t.Bucket.Size_ * nbuckets
            up := roundupsize(sz, !t.Bucket.Pointers())
            if up != sz {
                nbuckets = up / t.Bucket.Size_
            }
        }
        ...
    }
    ```

- 然后判断 `dirtyalloc` 指针，如果为空，则直接创建对应数组，可以看到正常的桶与溢出桶，是在一起进行内存分配的，其空间地址是连续的
- 如果 `dirtyalloc` 指针非空，则说明前置已进行过初始化，只需将其中的数据清空即可

    ```go
    func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
        ...
        if dirtyalloc == nil {
            buckets = newarray(t.Bucket, int(nbuckets))
        } else {
            buckets = dirtyalloc
            size := t.Bucket.Size_ * nbuckets
            if t.Bucket.Pointers() {
                memclrHasPointers(buckets, size)
            } else {12
                memclrNoHeapPointers(buckets, size)
            }
        }
        ...
    }
    ```

- 最终通过分配的数组长度来判断是否存在溢出桶，并分别返回普通桶的数组地址和溢出桶的数组地址
- 除此以外，还额外将最后一个溢出桶的 `overflow` 指针，指向了第一个普通桶，其他溢出桶的 `overflow` 指针均为 `nil`，这个差异会用于判断当前是否还有空余的溢出桶

    ```go
    func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
        ...
        if base != nbuckets {
            nextOverflow = (*bmap)(add(buckets, base*uintptr(t.BucketSize)))
            last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.BucketSize)))
            last.setoverflow(t, (*bmap)(buckets))
        }
        return buckets, nextOverflow
    }
    ```

## 数据处理

### 新增 & 修改数据

#### [mapassign](https://github.com/golang/go/blob/377646589d5fb0224014683e0d1f1db35e60c3ac/src/runtime/map.go#L614) 函数

新增或是修改数据，均通过 `m[k] = v` 的方式进行：

```go
m := map[string]int{"key_1": 1}
m["key_2"] = 2
m["key_1"] = 100
fmt.Println(m) // "map[key_1:100 key_2:2]"
```

在底层，则是通过 [mapassign](https://github.com/golang/go/blob/377646589d5fb0224014683e0d1f1db35e60c3ac/src/runtime/map.go#L614) 函数实现相关功能：

- 判断 key 是否已经存在，若存在则修改 value 值
- 若 key 不存在，则寻找可插入新数据的位置
- 若不存在可插入位置，则触发扩容或是新增溢出桶进行存储
- 最终返回 value 对象对应的指针，由函数调用方进行修改

函数详细介绍如下：

- 预处理：
  - 校验哈希表本身的初始化
  - 校验哈希表的并发写问题，哈希表本身不是线程安全的
  - 生成 key 对应的哈希值
  - 设置 `hashWriting` 标记位
  - 校验桶数组，若不存在则进行初始化

    ```go
    func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
        if h == nil {
            panic(plainError("assignment to entry in nil map"))
        }
        ...
        if h.flags&hashWriting != 0 {
            fatal("concurrent map writes")
        }
        hash := t.Hasher(key, uintptr(h.hash0))

        // Set hashWriting after calling t.hasher, since t.hasher may panic,
        // in which case we have not actually done a write.
        h.flags ^= hashWriting

        if h.buckets == nil {
            h.buckets = newobject(t.Bucket) // newarray(t.Bucket, 1)
        }
        ...
    }
    ```

- 确认哈希桶，声明或初始化一些关键变量
  - 桶号由哈希值的低 B 位来确认，`hash & (1<<B-1)` 最终的结果，等价于 `hash % 2^B`
  - 确认当前哈希表的扩容状态，优先确保扩容完成（后文详细介绍）
  - 通过桶号获取桶的地址
  - 计算高位的哈希值，并即确保哈希值的最小值，兼容标记位逻辑
  - 声明 key 值索引的指针 `inserti`
  - 声明 key 和 value 的指针 `insertk` 和 `elem`

    ```go
    func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
        ...
    again:
        bucket := hash & bucketMask(h.B)
        if h.growing() {
            growWork(t, h, bucket)
        }
        b := (*bmap)(add(h.buckets, bucket*uintptr(t.BucketSize)))
        top := tophash(hash)

        var inserti *uint8
        var insertk unsafe.Pointer
        var elem unsafe.Pointer
        ...
    }

    // bucketMask returns 1<<b - 1, optimized for code generation.
    func bucketMask(b uint8) uintptr {
        return bucketShift(b) - 1
    }

    // tophash calculates the tophash value for hash.
    func tophash(hash uintptr) uint8 {
        top := uint8(hash >> (goarch.PtrSize*8 - 8))
        if top < minTopHash {
            top += minTopHash
        }
        return top
    }
    ```

- 遍历哈希桶，优先寻找当前 key 和 value 对应的位置，否则寻找一个最靠前的可插入的位置
  - 若 `tophash` 不匹配，桶中当前位置为空，且 key 的索引的指针为空，则更新 key 与 value 相关指针，记录下该位置
  - 若 `tophash` 不匹配，桶中当前位置以及之后位置为空，则结束当前循环，否则继续进行循环
  - 若 `tophash` 匹配，则尝试匹配当前位置对应的 key 值，若不匹配，则继续进行循环
  - 若 `tophash` 匹配，且当前位置的 key 值也匹配，则说明匹配成功，更新 key 值，获取 value 的地址，并直接前往 `done` 所对应的代码块，最终返回 value 对应的指针（在函数外部进行更新）
  - 若当前哈希桶遍历结束（一个桶中最多 8 个元素），且仍存在溢出桶，则继续按照如上逻辑，遍历溢出桶，否则结束当前循环

    ```go
    func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
        ...
    bucketloop:
        for {
            for i := uintptr(0); i < abi.MapBucketCount; i++ {
                if b.tophash[i] != top {
                    if isEmpty(b.tophash[i]) && inserti == nil {
                        inserti = &b.tophash[i]
                        insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
                        elem = add(unsafe.Pointer(b), dataOffset+abi.MapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
                    }
                    if b.tophash[i] == emptyRest {
                        break bucketloop
                    }
                    continue
                }
                k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
                if t.IndirectKey() {
                    k = *((*unsafe.Pointer)(k))
                }
                if !t.Key.Equal(key, k) {
                    continue
                }
                // already have a mapping for key. Update it.
                if t.NeedKeyUpdate() {
                    typedmemmove(t.Key, k, key)
                }
                elem = add(unsafe.Pointer(b), dataOffset+abi.MapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
                goto done
            }
            ovf := b.overflow(t)
            if ovf == nil {
                break
            }
            b = ovf
        }
    }
    ```

- 若如上循环结束时，未找到已经存在的 key 值，则执行插入逻辑，判断扩容和溢出桶
  - 执行插入逻辑前，优先判断是否需要进行扩容，若需要，则执行扩容逻辑，此时桶号会发生变化（桶号为低 B 位，但是 B 会发生变化），所以要回到 `again` 处重新查找桶号以及所要插入的位置
  - 若不需要扩容，且没有找到合适的插入位置，则创建新的溢出桶，并更新 key 和 value 相关的指针为新的溢出桶的首位

    ```go
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
            // The current bucket and all the overflow buckets connected to it are full, allocate a new one.
            newb := h.newoverflow(t, b)
            inserti = &newb.tophash[0]
            insertk = add(unsafe.Pointer(newb), dataOffset)
            elem = add(insertk, abi.MapBucketCount*uintptr(t.KeySize))
        }
        ...
    }
    ```

- 执行数据插入逻辑
  - 判断 key 和 value 是否是间接引用，若是，则创建对应类型的新的对象，并将地址保存至所要插入的位置处，然后更新 key 的指针为实际存储 key 对应地址的指针（value 对应的指针在函数 `done` 代码块中会统一处理）
  - 更新 key 对应的值（最终会返回 value 对应的指针，在外部更新 value 的值）
  - 更新 key 的索引的值，即 `tophash` 的值
  - 修改哈希表中元素个数

    ```go
    func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
        ...
        // store new key/elem at insert position
        if t.IndirectKey() {2
            kmem := newobject(t.Key)
            *(*unsafe.Pointer)(insertk) = kmem
            insertk = kmem
        }
        if t.IndirectElem() {
            vmem := newobject(t.Elem)
            *(*unsafe.Pointer)(elem) = vmem
        }
        typedmemmove(t.Key, insertk, key)
        *inserti = top
        h.count++
        ...
    }
    ```

- 最终参数处理
  - 判断并发写冲突
  - 清除 `hashWriting` 状态位，标记写入完成
  - 更新 value 指针，最终返回 value 实际存储的地址对应的指针，用于外部修改 value

    ```go
    func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
        ...
    done:
        if h.flags&hashWriting == 0 {
            fatal("concurrent map writes")
        }
        h.flags &^= hashWriting
        if t.IndirectElem() {
            elem = *((*unsafe.Pointer)(elem))
        }
        return elem
    }
    ```

#### [newoverflow](https://github.com/golang/go/blob/13c49096fd3b08ef53742dd7ae8bcfbfa45f3173/src/runtime/map.go#L239) 函数

一个哈希桶中最多可以存放 8 个元素，当哈希冲突超过这个数值时，需要使用溢出桶来存放新增元素。当需要新增溢出桶时，会通过 [newoverflow](https://github.com/golang/go/blob/13c49096fd3b08ef53742dd7ae8bcfbfa45f3173/src/runtime/map.go#L239) 函数处理相关逻辑。

- 创建溢出桶
  - 优先判断哈希表中的 `extra.nextOverflow` 指针是否为空，若非空，则说明创建哈希表时，预分配的溢出桶还存在，可以直接使用
  - 若在使用了该溢出桶后，还有剩余，则动态更新 `extra.nextOverflow` 指针，指向下一个可用的溢出桶
  - 若当前溢出桶为预分配的最后一个，即 `overflow` 指针非空（初始化时，最后一个溢出桶的 `overflow` 指针会指向第一个普通桶，其他溢出桶的 `overflow` 指针为空），则将其 `overflow` 指针重新置为空值，用于后续链接其他溢出桶，并将哈希表中的 `extra.nextOverflow` 指针置为空值，表示当前预分配的溢出桶，已全部消耗
  - 若当前不存在可用的溢出桶，则直接创建

```go
func (h *hmap) newoverflow(t *maptype, b *bmap) *bmap {
    var ovf *bmap
    if h.extra != nil && h.extra.nextOverflow != nil {
        // We have preallocated overflow buckets available.
        // See makeBucketArray for more details.
        ovf = h.extra.nextOverflow
        if ovf.overflow(t) == nil {
            // We're not at the end of the preallocated overflow buckets. Bump the pointer.
            h.extra.nextOverflow = (*bmap)(add(unsafe.Pointer(ovf), uintptr(t.BucketSize)))
        } else {
            // This is the last preallocated overflow bucket.
            // Reset the overflow pointer on this bucket,
            // which was set to a non-nil sentinel value.
            ovf.setoverflow(t, nil)
            h.extra.nextOverflow = nil
        }
    } else {
        ovf = (*bmap)(newobject(t.Bucket))
    }
    ...
}

```

- 更新溢出桶相关配置
  - 更新哈希表的溢出桶计数，即 `noverflow` 字段，用于扩容等逻辑使用
  - 当哈希表中的 key 和 value 不包含指针时，额外修改 `extra.overflow` 字段，将该溢出桶添加进去，避免被 GC 回收
  - 最后将该溢出桶链接在原本的桶链表上，返回溢出桶指针

```go
func (h *hmap) newoverflow(t *maptype, b *bmap) *bmap {
    ...
    h.incrnoverflow()
    if !t.Bucket.Pointers() {
        h.createOverflow()
        *h.extra.overflow = append(*h.extra.overflow, ovf)
    }
    b.setoverflow(t, ovf)
    return ovf
}

func (h *hmap) createOverflow() {
    if h.extra == nil {
        h.extra = new(mapextra)
    }
    if h.extra.overflow == nil {
        h.extra.overflow = new([]*bmap)
    }
}
```

### 访问数据

### 删除数据

## 扩容
