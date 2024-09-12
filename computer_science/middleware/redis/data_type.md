# Data Type

在 Redis 的使用中，有五种常见的类型，即 `String`、`Hash`、`List`、`Set` 和 `Zset`，后续还额外支持了四种特殊场景会使用的类型 `BitMap`、`HyperLogLog`、`GEO` 和 `Stream`。通过丰富的数据类型，支持了不同的业务特征。

在底层的实现上，则分别对应了 SDS(Simple Dynamic String)、双向链表(linked list)、压缩列表(ziplist)、哈希表(hash table)、跳表(skiplist)、整数集合(intset)、快表(quicklist)、(listpack) 这几种数据结构。通过不同数据结构的选择，为上层的数据类型的提供了较好的读写性能。

对于具体的类型，比如 `Set`，在数据量较少时，会优先使用整数集合作为底层数据结构，数据量较大时，会使用哈希表作为底层数据结构

除此以外，Redis 还封装了对象类型，每一个键值对都分别对应了一个键对象和一个值对象。上述所提到的每一个类型，其实质都是对象类型，即 `String` 类型，其实是 `String` 对象。

可以通过对象中的类型字段，区分对象底层的真实类型，还可以通过对象中的编码字段，区分底层真实的数据结构。

## Object

`Object` 对于使用者来说是无感的，但却是 Redis 中最为核心的数据结构，每次在 Redis 中新创建一个键值对时，其实是创建了两个对象，一个用作 key，一个用作 value。

Redis 会根据用户使用的命令不同，以及真正输入的数据类型不同，最终决定对象的真实类型。

### 数据结构

每个对象都由一个 [redisObject(robj)](https://github.com/redis/redis/blob/7.0.0/src/server.h#L845) 结构体进行表示。字段含义如下：

- `type`：对象的类型，如 `String`、`Hash` 等
- `encoding`：对象的编码方式，即底层数据结构，如 `SDS`、`hash table` 等
- `lru`：最近访问记录
  - 对于 LRU（最近最少使用） 淘汰算法来说，记录 key 最后一次的访问时间戳
  - 对于 LFU（最近最不常用） 淘汰算法来说，高 16 位记录最后访问时间，低 8 位记录访问次数
- `refcount`：引用计数，用于 GC
- `ptr`：指向底层数据结构的指针

```C
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

对象类型与编码类型的对应关系，如下所示，在目前的版本中(7.0)，压缩列表(ziplist)被紧凑列表(listpack)取代了，列表对象底层也改为使用快表(quicklist)来实现。

<table>
    <tr>
        <th>对象类型</th>
        <th>数据结构</th>
        <th>编码类型</th>
    </tr>
    <tr>
        <td rowspan="3"><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L638">OBJ_STRING</a></td>
        <td>整数</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L826">OBJ_ENCODING_INT</a></td>
    </tr>
    <tr>
        <td>Embedded SDS</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L833">OBJ_ENCODING_EMBSTR</a></td>
    </tr>
    <tr>
        <td>SDS</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L825">OBJ_ENCODING_RAW</a></td>
    </tr>
    <tr>
        <td rowspan="3"><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L639">OBJ_LIST</a></td>
        <td>压缩列表（不再使用）</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L830">OBJ_ENCODING_ZIPLIST</a></td>
    </tr>
    <tr>
        <td>双向链表（不再使用）</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L829">OBJ_ENCODING_LINKEDLIST</a></td>
    </tr>
    <tr>
        <td>快表</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L834">OBJ_ENCODING_QUICKLIST</a></td>
    </tr>
    <tr>
        <td rowspan="2"><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L640">OBJ_SET</a></td>
        <td>整数集合</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L831">OBJ_ENCODING_INTSET</a></td>
    </tr>
    <tr>
        <td>哈希表</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L827">OBJ_ENCODING_HT</a></td>
    </tr>
    <tr>
        <td rowspan="3"><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L641">OBJ_ZSET</a></td>
        <td>压缩列表（不再使用）</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L830">OBJ_ENCODING_ZIPLIST</a></td>
    </tr>
    <tr>
        <td>紧凑列表</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L836">OBJ_ENCODING_LISTPACK</a></td>
    </tr>
    <tr>
        <td>跳表</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L832">OBJ_ENCODING_SKIPLIST</a></td>
    </tr>
    <tr>
        <td rowspan="3"><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L642">OBJ_HASH</a></td>
        <td>压缩列表（不再使用）</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L830">OBJ_ENCODING_ZIPLIST</a></td>
    </tr>
    <tr>
        <td>紧凑列表</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L836">OBJ_ENCODING_LISTPACK</a></td>
    </tr>
    <tr>
        <td>哈希表</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L827">OBJ_ENCODING_HT</a></td>
    </tr>
    <tr>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L656">OBJ_STREAM</a></td>
        <td>基数树</td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L835">OBJ_ENCODING_STREAM</a></td>
    </tr>
</table>

### 常用命令

```shell
redis> rpush list_key 1 2 3 4 5 # 创建列表对象
(integer) 5

redis> type list_key # 获取对象类型
"list"

redis> object encoding list_key # 获取对象编码类型
"listpack"
```

## String

`String` 类型是 Redis 中最为基础的一个基本类型，业务层的所有非容器类型的数据，例如整数、浮点数、字符串、二进制数据等，在 Redis 中均是以 `String` 类型进行存储，其编码方式可以是 `int`、`embstr` 或是 `raw`，其中 `embstr` 和 `rar`，底层的数据结构为 SDS((Simple Dynamic String))。

### SDS 数据结构

SDS 底层的数据结构会根据实际所需大小，动态决定，以 [sdshdr8](https://github.com/redis/redis/blob/7.0.0/src/sds.h#L51) 为例：

- `len`：字符串当前的长度
- `alloc`：分配的字符串的总长度，即不包括结构体头部以及尾部的空终止符
- `flags`：标志位
- `buf`：存储字符串内容的字符数据

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

出于节省的目的考虑，在 [sdshdr8](https://github.com/redis/redis/blob/7.0.0/src/sds.h#L51) 中，`len` 与 `alloc` 字段的类型均为 `uint8_t`，相对应的，[sdshdr16](https://github.com/redis/redis/blob/7.0.0/src/sds.h#L57)、[sdshdr32](https://github.com/redis/redis/blob/7.0.0/src/sds.h#L63) 和 [sdshdr64](https://github.com/redis/redis/blob/7.0.0/src/sds.h#L69) 中，`len` 与 `alloc` 字段的类型分别为 `uint16_t`、`uint32_t` 和 `uint64_t`。

不过在实际使用中，`String` 类型允许的最大长度，还远远不到 `uint64_t` 的范围，在 [checkStringLength()](https://github.com/redis/redis/blob/7.0.0/src/t_string.c#L40) 函数中，限制了最大的大小为 [proto_max_bulk_len](https://github.com/redis/redis/blob/7.0.0/src/config.c#L3046) 的大小，即 512MB

```c
standardConfig static_configs[] = {
    ...
    createLongLongConfig("proto-max-bulk-len", NULL, DEBUG_CONFIG | MODIFIABLE_CONFIG, 1024*1024, LONG_MAX, server.proto_max_bulk_len, 512ll*1024*1024, MEMORY_CONFIG, NULL, NULL), /* Bulk request max size */
    ...
}

static int checkStringLength(client *c, long long size) {
    if (!mustObeyClient(c) && size > server.proto_max_bulk_len) {
        addReplyError(c,"string exceeds maximum allowed size (proto-max-bulk-len)");
        return C_ERR;
    }
    return C_OK;
}
```

### 编码方式

### 常用命令

### 应用场景

## 参考

- 《Redis 设计与实现》（基于 Redis 3.0 版本）
- <https://xiaolincoding.com/redis/data_struct/command.html#string>
