# Data Type

在 Redis 的使用中，有五种常见的类型，即 `String`、`Hash`、`List`、`Set` 和 `Zset`，后续还额外支持了四种特殊场景会使用的类型 `BitMap`、`HyperLogLog`、`GEO` 和 `Stream`。通过丰富的数据类型，支持了不同的业务特征。

在底层的实现上，则分别对应了 SDS(Simple Dynamic String)、双向链表(linked list)、压缩列表(ziplist)、哈希表(hash table)、跳表(skiplist)、整数集合(intset)、快表(quicklist)、紧凑列表(listpack) 这几种数据结构。通过不同数据结构的选择，为上层的数据类型的提供了较好的读写性能。

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
- `lru`：最近访问记录，[LRU_BITS](https://github.com/redis/redis/blob/7.0.0/src/server.h#L838) 值为 24
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

<br>

对象类型与编码类型的对应关系，如下所示，在目前的版本中(7.0)，压缩列表(ziplist)被紧凑列表(listpack)取代了，列表对象底层也改为使用快表(quicklist)来实现。

<table>
    <tr>
        <th>对象类型</th>
        <th>编码类型</th>
        <th>数据结构</th>
    </tr>
    <tr>
        <td rowspan="3"><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L638">OBJ_STRING</a></td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L826">OBJ_ENCODING_INT</a></td>
        <td>整数</td>
    </tr>
    <tr>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L833">OBJ_ENCODING_EMBSTR</a></td>
        <td>Embedded SDS</td>
    </tr>
    <tr>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L825">OBJ_ENCODING_RAW</a></td>
        <td>SDS</td>
    </tr>
    <tr>
        <td rowspan="3"><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L639">OBJ_LIST</a></td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L830">OBJ_ENCODING_ZIPLIST</a></td>
        <td>压缩列表（不再使用）</td>
    </tr>
    <tr>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L829">OBJ_ENCODING_LINKEDLIST</a></td>
        <td>双向链表（不再使用）</td>
    </tr>
    <tr>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L834">OBJ_ENCODING_QUICKLIST</a></td>
        <td>快表</td>
    </tr>
    <tr>
        <td rowspan="2"><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L640">OBJ_SET</a></td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L831">OBJ_ENCODING_INTSET</a></td>
        <td>整数集合</td>
    </tr>
    <tr>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L827">OBJ_ENCODING_HT</a></td>
        <td>哈希表</td>
    </tr>
    <tr>
        <td rowspan="3"><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L641">OBJ_ZSET</a></td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L830">OBJ_ENCODING_ZIPLIST</a></td>
        <td>压缩列表（不再使用）</td>
    </tr>
    <tr>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L836">OBJ_ENCODING_LISTPACK</a></td>
        <td>紧凑列表</td>
    </tr>
    <tr>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L832">OBJ_ENCODING_SKIPLIST</a></td>
        <td>跳表</td>
    </tr>
    <tr>
        <td rowspan="3"><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L642">OBJ_HASH</a></td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L830">OBJ_ENCODING_ZIPLIST</a></td>
        <td>压缩列表（不再使用）</td>
    </tr>
    <tr>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L836">OBJ_ENCODING_LISTPACK</a></td>
        <td>紧凑列表</td>
    </tr>
    <tr>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L827">OBJ_ENCODING_HT</a></td>
        <td>哈希表</td>
    </tr>
    <tr>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L656">OBJ_STREAM</a></td>
        <td><a href="https://github.com/redis/redis/blob/7.0.0/src/server.h#L835">OBJ_ENCODING_STREAM</a></td>
        <td>基数树</td>
    </tr>
</table>

### 常用命令

```shell
> rpush list_key 1 2 3 4 5 # 创建列表对象
(integer) 5

> type list_key # 获取对象类型
"list"

> object encoding list_key # 获取对象编码类型
"listpack"
```

## String

`String` 类型是 Redis 中最为基础的一个基本类型，业务层的所有非容器类型的数据，例如整数、浮点数、字符串、二进制数据等，在 Redis 中均是以 `String` 类型进行存储。

其编码方式可以是 `int`、`embstr` 或是 `raw`，其中 `embstr` 和 `rar`，底层的数据结构为 SDS(Simple Dynamic String)。

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

<br>

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

<br>

相较于 C 语言原生字符串，SDS 主要由如下几个优势：

- 读取长度时，直接读取结构体的属性 `len`，时间复杂度为 O(1)
- 读取数据时，根据 `len` 来判断当前所要读取的长度，而非空白符，所以可以存储包含空白符的数据，即可以存储所有二进制数据
- 写入数据时，可以根据 `len` 和 `alloc` 两个关键字判断空间是否足够，不够的话还可以修改底层缓冲区数据，动态扩容

### 编码方式

- `int`：当存储的 value 是整数，且可以用 `long` 类型进行表示时，此时会直接将 [redisObject](https://github.com/redis/redis/blob/7.0.0/src/server.h#L845) 中的 `ptr`，从 `void *` 类型转为 `long` 类型，用于存储 value。

  - `void *` 与 `long` 占用的字节数始终相同，在 32 位机器为 $4$ 字节，在 64 位机器为 $8$ 字节

  - 此时结构体如下所示：

    <table style="width:100%; text-align:center;">
        <tr>
            <th colspan="3">redisObject</th>
        </tr>
        <tr>
            <td>type</td>
            <td>encoding</td>
            <td>ptr</td>
        </tr>
        <tr>
            <td>string</td>
            <td>int</td>
            <th>value</th>
        </tr>
    </table>

<br>

- `embstr`：当存储的 value 是字符串，且字符长度小于 [OBJ_ENCODING_EMBSTR_SIZE_LIMIT](https://github.com/redis/redis/blob/7.0.0/src/object.c#L119) 时，会采用 `embstr` 编码，针对 [redisObject](https://github.com/redis/redis/blob/7.0.0/src/server.h#L845) 与 [sdshdr8](https://github.com/redis/redis/blob/7.0.0/src/sds.h#L51) 仅进行一次内存分配，提高性能。

  - 在实际使用中，value 为大整数、浮点数、二进制数据等情况，均为被统一转为字符串来存储

  - 在 7.0 版本中，[OBJ_ENCODING_EMBSTR_SIZE_LIMIT](https://github.com/redis/redis/blob/7.0.0/src/object.c#L119) 的值为 44
  - 在 [createStringObject()](https://github.com/redis/redis/blob/7.0.0/src/object.c#L120) 函数中，会实现上述决策逻辑

    ```c
    #define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
    robj *createStringObject(const char *ptr, size_t len) {
        if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
            return createEmbeddedStringObject(ptr,len);
        else
            return createRawStringObject(ptr,len);
    }
    ```

<br>

- `raw`：在其他情况下，均会使用 `raw` 编码类型，先分配 [redisObject](https://github.com/redis/redis/blob/7.0.0/src/server.h#L845) 的内存，再分配 [sdshdr8](https://github.com/redis/redis/blob/7.0.0/src/sds.h#L51) 的内存
  - 对于 `embstr` 来说，创建与释放均仅需要调用一次函数，且内存连续可以更好的利用 CPU 缓存，
  - 字符串长度在增长时，有可能会触发扩容逻辑，重新进行内存分配，有可能会不满足 `int` 和 `embstr` 的条件，出于简单考虑，规定 `int` 和 `embstr` 编码方式仅支持读，触发写逻辑时会将其转为 `raw` 编码类型再进行修改

<br>

- `size_limit`

  - 对于 [redisObject](https://github.com/redis/redis/blob/7.0.0/src/server.h#L845) 来说，在 64 位系统下，共占用 $16$ 字节

    <table style="width:100%; text-align:center;">
        <tr>
            <th></th>
            <th colspan="5">redisObject</th>
        </tr>
        <tr>
            <th>field</th>
            <td>type</td>
            <td>encoding</td>
            <td>lru</td>
            <td>refcount</td>
            <td>ptr</td>
        </tr>
        <tr>
            <th>type</th>
            <td>4 bit</td>
            <td>4 bit</td>
            <td>24 bit</td>
            <td>int</td>
            <td>void *</td>
        </tr>
        <tr>
            <th>byte</th>
            <td>0.5</td>
            <td>0.5</td>
            <td>3</td>
            <td>4</td>
            <td>8</td>
        </tr>
    </table>

  - 而对于 [sdshdr8](https://github.com/redis/redis/blob/7.0.0/src/sds.h#L51) 来说，在不考虑 `buf` 字段的前提下，共占用 $3$ 字节，此时假定 value 的长度为 $n$，则 buf 的长度为 value 加上空白符的长度即 $n + 1$ 字节

    <table style="width:100%; text-align:center;">
        <tr>
            <th></th>
            <th colspan="4">sdshdr8</th>
        </tr>
        <tr>
            <th>field</th>
            <td>len</td>
            <td>alloc</td>
            <td>flag</td>
            <td>buf</td>
        </tr>
        <tr>
            <th>type</th>
            <td>uint8_t</td>
            <td>uint8_t</td>
            <td>unsigned char</td>
            <td>char [ ]</td>
        </tr>
        <tr>
            <th>byte</th>
            <td>1</td>
            <td>1</td>
            <td>1</td>
            <td>n+1</td>
        </tr>
    </table>

  - 从内存分配和内存对齐的角度来说，64 位系统下，一次分配的内存为 $16$ 字节的倍数，且是 $2$ 的整数次幂，即 $16$ 字节、$32$ 字节、$64$ 字节，等等
  - 上述两个结构体占用的字节数为 $20 + n$，故较为合适的分配方式，是直接分配 $64$ 字节（$32$ 字节太小），此时 $n$ 的最大值即为 $64-20=44$

### 常用命令

- 基础操作：

    ```shell
    > set int_key 100 # 设置整数，key 值重复时会直接覆盖
    "OK"

    > get int_key # 获取 value，对于整数来说，最终返回的也是字符串类型
    "100"

    > append int_key _append # 附加新字符串，对于整数，会将其转为 raw 类型再修改
    (integer) 10             # 返回修改后的字符串长度

    > get int_key
    "100_append"

    > exists int_key # 判断 key 是否存在
    (integer) 1

    > strlen int_key # 获取字符串长度
    (integer) 10

    > DEL int_key # 删除
    (integer) 1
    ```

<br>

- range 操作

    ```shell
    > set string_key string_value # 设置字符串
    "OK"

    > setrange string_key 12 _edit # 从指定位置开始修改字符串
    (integer) 17

    > get string_key
    "string_value_edit"

    > setrange string_key 20 _out_of_range # 越界时，会自动扩容，并填充空白符 \x00
    (integer) 33

    > get string_key
    "string_value_edit\x00\x00\x00_out_of_range"

    > getrange string_key 0 11 # 获取指定索引内的数据
    "string_value"
    ```

<br>

- 批量操作

    ```shell
    > mset key1 value1 key2 value2 # 批量设置
    "OK"

    > mget key1 key2 # 批量读取
    1) "value1"
    2) "value2"
    ```

<br>

- 计数器

    ```shell
    > set int_key 100
    "OK"

    > incr int_key # 将整数类型的值加一，并返回整数结果
    (integer) 101

    > incrby int_key 20 # 将整数类型的值加任意值，并返回整数结果
    (integer) 121

    > decr int_key # 将整数类型的值减一，并返回整数结果
    (integer) 120

    > decrby int_key 20 # 将整数类型的值减任意值，并返回整数结果
    (integer) 100
    ```

<br>

- 过期时间

    ```shell
    > set expire_key value ex 3600 # 设置键值对时，设置过期时间
    "OK"

    > setex expire_key2 3600 value # 设置键值对时，设置过期时间
    "OK"

    > expire expire_key 600 # 更新过期时间（要求 key 已存在）
    (integer) 1

    > ttl expire_key # 查看过期时间
    (integer) 593

    > ttl expire_key2 # 查看过期时间
    (integer) 3571
    ```

<br>

- 不存在时插入

    ```shell
    > setnx nx_key value # 不存在时插入，结果为 1 代表插入成功
    (integer) 1

    > setnx nx_key value # 不存在时插入，结果为 0 代表已经存在
    (integer) 0

    > set nx_key2 value nx # 不存在时插入，"OK" 代表插入成功，可结合其他关键字使用，比如过期时间
    "OK"

    > set nx_key2 value nx # 不存在时插入，nil 代表已经存在
    (nil)   
    ```

### 应用场景

- 分布式存储：在分布式架构下，某些临时数据需要维护在服务器内存，而不需要持久化存储时，可以通过 Redis 实现数据共享

<br>

- 缓存对象：常使用对象标识加主键一起作为 key 值

  - 缓存 json 字符串，由业务层处理序列化/反序列化逻辑

    ```shell
    > set user:1 '{"name":"silk", "age":18}'
    "OK"
    ```

  - 分属性进行缓存，避免了序列化开销，但是需要进行数据组装

    ```shell
    > mset user:1:name silk user:1:age 18
    "OK"
    ```

<br>

- 计数器：Redis 本身在处理命令时是单线程操作，不存在并发问题，所以适合于计数场景，比如页面 PV、点赞次数等

<br>

- 分布式锁：
  - 通过 `nx` 命令，可以实现加锁操作，插入成功代表加锁成功

    ```shell
    setnx lock_key unique_value
    ```

  - 通过设置过期时间，可以实现超时控制

    ```shell
    set lock_key unique_value nx ex 10
    ```

  - 通过删除 key 值，可以实现解锁操作，但是必须通过 lua 脚本，判断加锁、解锁来自同一个实例，然后再删除 key 值，确保操作的原子性

    ```lua
    if redis.call("get",KEYS[1]) == ARGV[1] then
        return redis.call("del",KEYS[1])
    else
        return 0
    end
    ```

## List

`List` 是最常见的容器类型之一，内部元素按照特定顺序进行排列，在使用时，可以从头部或尾部插入元素。`List` 内部的元素类型，均为 `String` 类型。

在老版本中，如果 `List` 中元素个数较少，且每个元素都小于特定值，会使用压缩列表(ziplist)作为底层数据结构，其他情况会使用双向链表(linked list)作为底层数据结构。

在 3.2 版本以后，`List` 仅会使用快表(quicklist)这一种数据结构。

### 常用命令

```shell
> rpush list_key r1 # 向队尾(right)插入一个元素，首次插入会新建列表元素
(integer) 1

> lpush list_key l2 l3 # 向队首(left)按序插入多个元素，即后插入的元素在左侧
(integer) 3

> rpush list_key r4 r5 # 向队尾(right)按序插入多个元素，即后插入的元素在右侧
(integer) 5

> lrange list_key 0 -1 # 打印列表内所有元素
1) "l3"
2) "l2"
3) "r1"
4) "r4"
5) "r5"

> lrange list_key 3 99 # 打印区间内的所有元素，无越界问题
1) "r4"
2) "r5"

> lrange list_key 98 99 # 打印区间内的所有元素，无越界问题
(empty list or set)

> lpop list_key # 从队首(left)弹出一个元素
"l3"

> rpop list_key # 从队尾(right)弹出一个元素
"r5"

> lrange list_key 0 99
1) "l2"
2) "r1"
3) "r4"

> blpop list_key 10 # 从队首(left)弹出一个元素，最多阻塞等待 timeout 秒
1) "list_key"
2) "l2"

> brpop list_key 10 # 从队尾(right)弹出一个元素，最多阻塞等待 timeout 秒
1) "list_key"
2) "r4"

> rpush list_key v5 v6 v7
(integer) 4

> rpoplpush list_key list_key2 # 从列表一的队尾弹出一个元素，返回并同时添加至列表二的队首
"v7"

> brpoplpush list_key list_key2 10 # 最多阻塞等待 timeout 秒
"v6"
```

### 应用场景

- 消息队列

  - 消息队列是一种异步通信方式，在实现时，需要保障消息有序、不重复、不丢失。

    <br>

  - 有序：`List` 支持从队首或队尾进行读写，在使用时，控制生产者、消费者固定从不同的方向操作即可
    - 在消费者读取消息时，可以通过阻塞式命令来读取，例如 `brpop`，避免轮询操作

    <br>

  - 不重复：`List` 本身没有提供相关能力保障，需要业务层自行为每条消息生成一个全局 ID，自行进行保障
    - 比如使用 `setnx` 命令维护消费记录

    <br>

  - 不丢失：`List` 本身仅提供了消息的入队和出队能力，对于消息出队之后，但是因为网络或业务层问题导致消费行为中断的场景，没有办法进行保障
    - 从数据交互层面进行分析，其根本原因是缺少了 ack 机制
    - 所以可以将出队的数据，存储在另外一个队列中，当消费行为完成后，再将数据最终出队，完成删除操作
    - 通过 `rpoplpush` 和 `brpoplpush` 命令，可以完成上述操作，且过程是原子的

    <br>

  - 消息模型：消息队列有两种常见模型，点对点模型(P2P)与发布订阅模型(Pub/Sub)，但是 `List` 仅具备单播能力，不具备广播能力，即无法实现发布订阅模型

## Hash

`Hash` 类型是典型的 kv 存储容器，可以通过 hash 算法，对 key 实现分组操作，进而快速找到指定 key 值所对应的 value 元素，Redis 本身键值对的存储结构，也是依赖于哈希表。

在老版本中，如果 `Hash` 中元素个数较少，且每个元素都小于特定值，会使用压缩列表(ziplist)作为底层数据结构，其他情况会使用哈希表(hash table)作为底层数据结构。

在 7.0 版本中，压缩列表(ziplist)被彻底废弃，改由紧凑列表(listpack)来实现。

### 常用命令

```shell
> hset hash_key field1 10  # 向指定哈希表中设置新元素，哈希表不存在则自动创建
(integer) 1

> hset hash_key field1 100 # 向指定哈希表中设置新元素，filed 重复则更新
(integer) 0

> hget hash_key field1 # 获取哈希表中指定元素
"100"

> hmset hash_key field2 20 field3 30 # 向指定哈希表中批量设置新元素
"OK"

> hmget hash_key field2 field3 # 批量获取哈希表中指定元素
1) "20"
2) "30"

> hdel hash_key field2 # 删除哈希表中指定元素
(integer) 1

> hlen hash_key # 获取指定哈希表的长度
(integer) 2

> hgetall hash_key # 获取指定哈希表中所有数据
1) "field1"
2) "100"
3) "field3"
4) "30"

> hincrby hash_key field1 10 # 将哈希表中指定整数元素添加指定增量
(integer) 110
```

### 应用场景

- 缓存对象：常使用对象标识加主键一起作为 key 值

  - 直接将缓存对象当作 `Hash` 类型来存储，对象的属性名对应于 `Hash` 对象的 field 部分

    ```shell
    > hmset user:1 name silk age 18
    "OK"

    > hgetall user:1
    1) "name"
    2) "silk"
    3) "age"
    4) "18"
    ```

  - 与 `String` 类型对比
    - `String`

        <table style="width:100%; text-align:center;">
            <tr>
                <th>Type</th>
                <th>Key</th>
                <th>value</th>
            </tr>
            <tr>
                <th rowspan="2">String</th>
                <td>user:1:name</td>
                <td>silk</td>
            </tr>
            <tr>
                <td>user:1:age</td>
                <td>18</td>
            </tr>
        </table>

    - `Hash`

        <table style="width:100%; text-align:center;">
            <tr>
                <th>Type</th>
                <th>Key</th>
                <th>field</th>
                <th>value</th>
            </tr>
            <tr>
                <th rowspan="3">Hash</th>
                <td rowspan="3">user:1</td>
            </tr>
            <tr>
                <td>name</td>
                <td>silk</td>
            </tr>
            <tr>
                <td>age</td>
                <td>18</td>
            </tr>
        </table>

## Set

## Zset

## BitMap

## HyperLogLog

## GEO

## Stream

## 参考

- 《Redis 设计与实现》（基于 Redis 3.0 版本）
- <https://xiaolincoding.com/redis/data_struct/command.html#string>
