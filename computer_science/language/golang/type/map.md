# 哈希表

哈希表是一系列键值对的无序集合，维护了 key 和 value 的映射关系，还提供了常数时间复杂度的读写性能。

在 Golang 中，`map` 类型实质上就是哈希表。数据结构 [hamp](https://github.com/golang/go/blob/e3d87d19320001e6081449550292d76ef660ab03/src/runtime/map.go#L109) 如下所示：

```go
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
