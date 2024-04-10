# 基础类型

## 整数

整数类型可以分为如下几种：

1. 明确声明了大小的类型，其中有符号整数用最高位 bit 来表示正负，一个 `n-bit` 有符号整数对应的数域为 $-2^{n-1}$ 到 $2^{n-1}-1$，无符号整数对应的数域为 是 0 到 $2^n-1$：
    | Size | Signed | Unsigned |
    | :-: | :-: | :-: |
    | 8 bits | int8 | uint8 |
    | 16 bits | int16 | uint16 |
    | 32 bits | int32 | uint32 |
    | 64 bits | int64 | uint64 |

2. 针对于 CPU 平台的类型，`int` 和 `uint`，可能是 32 bits 或者 64 bits。

3. Unicode 字符所使用的 `rune` 类型，与 `int32` 等价，需要 4 bytes 来支持所有字符。`byte` 类型与 `uint8` 等价，但是 `byte` 一般用作数据的单位，而非用来表示具体数值。

4. 用于存储指针数值的无符号整型 `uintptr`，大小同样取决于操作系统的位数，但是足以容纳指针。

需要注意的是，各种类型虽然存在等价关系，但是在实际使用中，各类型间仍然是截然不同的，例如不能将一个 `int32` 类型的值与 `int16` 的值相加，此时必须进行类型强转：

```go
var apples int32 = 1
var oranges int16 = 2
var compote int = apples + oranges // compile error
var compote = int(apples) + int(oranges)
```

## 浮点数

浮点数包括 `float32` 与 `float64` 两种，符合 IEEE754 浮点数国际标准定义。

浮点数能够表示的数值范围很大，其中 `float32` 的数域约为 1.4e-45 至 3.4e38，`float64` 的范围约为 4.9e-324 至 1.8e308，但是受限于有效 bit，`float32` 的精度约为 6 个十进制数，`float64` 的精度约为 15 个十进制数。

## 字符串

字符串是一个不可变的字节序列，通常也会被认为是采用了 UTF-8 编码的 Unicode 码点(`rune`)序列，即可通过 `len` 函数获取字符串中的**字节数目**。
> 注意 `len` 函数返回的不是字符数目，在填充的字符为非 ASCII 字符时，每个字符会占据多个字节，此时第 i 个字节不一定是第 i 个字符。

### 字符串索引

可通过索引操作 `s[i]` 返回第 i 个字节的字节值，注意索引不能越界，即 $0 \leq i < len(s)$

```go
s := "Hello, world!"
fmt.Println(len(s)) // "12"
fmt.Println(s[0], s[7]) // "72 119" ('H' and 'w')
```

也可通过 `s[i:j]` 生成 s 第 i 个字节到第 j 个字节（不包含 j）的子串，满足 $0 \leq i \leq j \leq len(s)$。当没有指定 i 或者 j 时，默认值分别为 `0` 和 `len(s)`。

```go
s := "Hello, world!"
fmt.Println(s[:5]) // "Hello"
fmt.Println(s[7:]) // "world!"
fmt.Println(s[:]) // "Hello, world!" 
```

值得注意的是，字符串本身是不可变的，所以当我们复制字符串或者使用字符串切片的子串时，共享底层数据的操作，是安全的，也没有必要分配新的内存。

```go
s := "Hello, world!"
sHeader := (*reflect.StringHeader)(unsafe.Pointer(&s))
fmt.Printf("%p\n", unsafe.Pointer(sHeader.Data)) // 0x49cccb

s0 := s[:4]
s0Header := (*reflect.StringHeader)(unsafe.Pointer(&s0))
fmt.Printf("%p\n", unsafe.Pointer(s0Header.Data)) // 0x49cccb (与s一致)

s1 := s[1:4]
s1Header := (*reflect.StringHeader)(unsafe.Pointer(&s1))
fmt.Printf("%p\n", unsafe.Pointer(s1Header.Data)) // 0x49cccc (偏移量为 c-b=1)

s2 := s[2:4]
s2Header := (*reflect.StringHeader)(unsafe.Pointer(&s2))
fmt.Printf("%p\n", unsafe.Pointer(s2Header.Data)) // 0x49cccd (偏移量为 d-c=1)

t := s
tHeader := (*reflect.StringHeader)(unsafe.Pointer(&t))
fmt.Printf("%p\n", unsafe.Pointer(tHeader.Data)) // 0x49cccb (与s一致)
```
