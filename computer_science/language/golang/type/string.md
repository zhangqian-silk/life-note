# 字符串

字符串是一个不可变的字节序列，通常也会被认为是采用了 UTF-8 编码的 Unicode 码点(`rune`)序列，即可通过 `len` 函数获取字符串中的**字节数目**。
> 注意 `len` 函数返回的不是字符数目，在填充的字符为非 ASCII 字符时，每个字符会占据多个字节，此时第 i 个字节不一定是第 i 个字符。

## 底层结构

> <https://go.dev/src/runtime/string.go>
> <https://go.dev/src/reflect/value.go>
> <https://go.dev/src/unsafe/unsafe.go>

字符串的底层数据结构包含了指向字节数组的指针和数组长度，在 `runtime` 包中，可以看到类似的内部使用的 `string` 的结构体 [stringStruct](https://github.com/golang/go/blob/960fa9bf66139e535d89934f56ae20a0e679e203/src/runtime/string.go#L232)：

```go
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```

而在 `reflect` 包中，可以看到 `string` 结构体在运行时的表现形式 [stringHeader](https://github.com/golang/go/blob/960fa9bf66139e535d89934f56ae20a0e679e203/src/reflect/value.go#L2832)：

```go
// StringHeader is the runtime representation of a string.
// ...
// Deprecated: Use unsafe.String or unsafe.StringData instead.
type StringHeader struct {
    Data uintptr
    Len  int
}
```

在老版本中，借助于上面暴露出的结构体，开发者可以实现 `string` 与 `slice` 的零拷贝的转换，但是因为 `Data` 的实际类型为 `uintprt`，没有任何类型校验，往往会导致非预期的行为产生，在高版本（Go 1.20）中，上述结构体被标记废弃，并在 `unsafe` 包中提供了类似能力，但是提供了类型安全的构建 `sting` 的方法 [String](https://github.com/golang/go/blob/960fa9bf66139e535d89934f56ae20a0e679e203/src/unsafe/unsafe.go#L262)：

```go
// String returns a string value whose underlying bytes
// start at ptr and whose length is len.
// ...
// Since Go strings are immutable, the bytes passed to String
// must not be modified afterwards.
func String(ptr *byte, len IntegerType) string
```

需要注意的是，在 Go 中，字符串是不允许被修改的，只允许重新赋值。这种设计保障了多线程操作以及底层字符数组数据共享时的安全性，而数据共享一方面节约了内存，另一方面在字符串复制以及获取子串时，能带来更高的性能。

在如下代码中，我们声明了字符串 `s`，并将其赋值给了字符串 `t`，可以发现他们在底层的字符数据上是共享的。当我们修改了 `s` 时，字符数组的首地址发生了改变，但是字符串 `t` 没有任何改变，可以证明在修改字符串时，实质是修改字符数组的指针，指向另外一处的字符数组，并非字符数组本身发生了改变。

```go
s := "Hello, world!"
fmt.Printf("%p\n", unsafe.StringData(s)) // 0x6c1b55

t := s
fmt.Printf("%p\n", unsafe.StringData(t)) // 0x6c1b55 (与s一致)

s = "Hello"
fmt.Printf("s:%s, %p", s, unsafe.StringData(s)) // s:Hello, 0x6c0af2
fmt.Printf("t:%s, %p", t, unsafe.StringData(t)) // t:Hello, world!, 0x6c1b55
```

## 字面量

字符串的字面量支持双引号与反引号两种方式：

- 前者被称作解释字符串（Interpreted string literals）与其他语言类似，用于声明单行字符串，对于特殊字符，需要转义处理。适合需要使用转义字符的场景。

- 后者被称作原生字符串（Raw string literals），则没有转义操作，字符串内容会被原样输出，可以包括任意字符（除了反引号本身）。适合包含大量特殊字符或多行文本，需要原样输出的场景，例如正则表达式，网址链接，json 字符串等。

例如，在需要换行操作时，解释字符串需要拼接 `\n` 来实现效果，原生字符串直接换行即可；在声明 json 字符串时，解释字符串需要对特殊字符 `"` 进行转义，原生字符串同样直接声明即可：

```go
s := "Hello\nworld!"
t := `Hello
world!`

s = "{\"key\": \"value\"}"
t = `{"key": "value"}`
```

### 解析方式

在扫描器中的 [next()](https://github.com/golang/go/blob/960fa9bf66139e535d89934f56ae20a0e679e203/src/cmd/compile/internal/syntax/scanner.go#L88) 函数中，可以看到编译器会针对于双引号和反引号，分别调用不同的字符串的处理逻辑：

```go
func (s *scanner) next() {
    ...
    switch s.ch {
    ...
    case '"': 
        s.stdString()

    case '`': 
        s.rawString()
    ...
    }
    ...
}
```

在 [stdString()](https://github.com/golang/go/blob/960fa9bf66139e535d89934f56ae20a0e679e203/src/cmd/compile/internal/syntax/scanner.go#L674) 函数中，会循环读取后续所有字符，直至遇到下一个双引号，期间如果遇到转义字符会进行特殊处理：

```go
func (s *scanner) stdString() {
    ok := true
    s.nextch()

    for {
        if s.ch == '"' {
            s.nextch()
            break
        }
        if s.ch == '\\' {
            s.nextch()
            if !s.escape('"') {
                ok = false
            }
            continue
        }
        if s.ch == '\n' {
            s.errorf("newline in string")
            ok = false
            break
        }
        if s.ch < 0 {
            s.errorAtf(0, "string not terminated")
            ok = false
            break
        }
        s.nextch()
    }

    s.setLit(StringLit, ok)
}
```

在 [rawString()](https://github.com/golang/go/blob/960fa9bf66139e535d89934f56ae20a0e679e203/src/cmd/compile/internal/syntax/scanner.go#L706C19-L706C30) 函数中，不会做任何额外的校验逻辑，会读取两个反引号间的所有内容，最终最终字符串的值：

```go
func (s *scanner) rawString() {
    ok := true
    s.nextch()

    for {
        if s.ch == '`' {
            s.nextch()
            break
        }
        if s.ch < 0 {
            s.errorAtf(0, "string not terminated")
            ok = false
            break
        }
        s.nextch()
    }
    // We leave CRs in the string since they are part of the
    // literal (even though they are not part of the literal
    // value).

    s.setLit(StringLit, ok)
}
```

## 索引

可通过索引操作 `s[i]` 返回第 i 个字节的字节值，注意：

  1. 索引不能越界，即 $0 \leq i < len(s)$；
  2. 字符串不能修改。

```go
s := "Hello, world!"
fmt.Println(len(s)) // "12"
fmt.Println(s[0], s[7]) // "72 119" ('H' and 'w')
s[0] = "h" // compile error: cannot assign to s[0]
```

也可通过 `s[i:j]` 生成 s 第 i 个字节到第 j 个字节（不包含 j）的子串，满足 $0 \leq i \leq j \leq len(s)$。当没有指定 i 或者 j 时，默认值分别为 `0` 和 `len(s)`。同样得益于字符串不允许被修改的特性，所有子串均可以安全地共享相同的字符数组，避免了额外的开销：

```go
s := "Hello, world!"
fmt.Printf("s:%s, %p\n", s, unsafe.StringData(s)) // s:Hello, world!, 0x281b56

s0 := s[:4]
fmt.Printf("s[:4]:%s, %p\n", s0, unsafe.StringData(s0)) // s[:4]:Hell, 0x281b56 (与s一致)

s1 := s[1:4]
fmt.Printf("s[1:4]:%s, %p\n", s1, unsafe.StringData(s1)) // s[1:4]:ell, 0x281b57 (偏移量为 57-56=1)

s2 := s[2:4]
fmt.Printf("s[2:4]:%s, %p\n", s2, unsafe.StringData(s2)) // s[2:4]:ll, 0x281b58 (偏移量为 58-56=2)
```

## 运算

字符串默认支持了一些运算符，即可以使用 `<`, `<=`, `>`, `>=` 和 `==` 对字符串逐字节进行比较：

```go
s := "abcde"
t := "bbcde"
fmt.Println(s <= t) // true
```

也支持使用 `+` 来进行字符串拼接：

```go
s := "Hello"
s += ", world!"
fmt.Println(s) // Hello, world!
```

## Byte Slice

字符串的不可变性，保障了安全和性能，但是当我们需要一些字符串的拼接等逻辑时，则会多次触发字符串的赋值，导致了没必要的内存开销。

例如我们需要针对于一个整数字符串，每隔三个字符插入一个逗号分隔符，在仅使用 `string` 类时，每次循环都需要构建新的 `string` 结构体：

```go
func addComma(s string) string {
    n := len(s)
    if n <= 3 {
        return s
    }

    commaIndex := n % 3
    if commaIndex == 0 {
        commaIndex = 3
    }

    result := s[:commaIndex]
    for i := commaIndex; i < n; i += 3 {
        result += "," + s[i:i+3]
    }

    return result
}
```

相比之下，我们可以直接使用字节数组，即 `[]byte` 来满足我们修改字符串的需求，在 json 序列化等场景广泛使用。

使用 `[]byte` 重写以上功能，在每次调用 `append()` 方法进行字符拼接操作时，如果切片本身的空间足够大，则不会触发任何额外的内存分配，且我们可以更前置的计算所需空间来初始化一个较为合理的切片大小：

```go
func addComma(s string) string {
    n := len(s)
    if n <= 3 {
        return s
    }

    result := make([]byte, 0, n+n/3)

    r := n % 3
    if r == 0 {
        r = 3
    }

    result = append(result, s[:r]...)
    for i := r; i < n; i += 3 {
        result = append(result, ',')
        result = append(result, s[i:i+3]...)
    }

    return string(result)
}
```

### 转换

从底层数据结构上讲，字符串与字节切片底层都是字节数组，故而他们之间可以方便地相互转换。但是可变与不可变的限制，会导致产生了一些额外的内存消耗。

```go
s := "hello"
b := []byte(s)
t := string(b)
```

当从字节数组转化至字符串时，需要构造一个字节数组的拷贝，用来确认字符串是只读的。[slicebytetostring()](https://github.com/golang/go/blob/8960925ad8dd1ef234731d94ebbea263e35a3e42/src/runtime/string.go#L81) 方法如下所示：

```go
func slicebytetostring(buf *tmpBuf, ptr *byte, n int) string {
    if n == 0 {
        return ""
    }
    if raceenabled {
        racereadrangepc(unsafe.Pointer(ptr),
            uintptr(n),
            getcallerpc(),
            abi.FuncPCABIInternal(slicebytetostring))
    }
    if msanenabled {
        msanread(unsafe.Pointer(ptr), uintptr(n))
    }
    if asanenabled {
        asanread(unsafe.Pointer(ptr), uintptr(n))
    }
    if n == 1 {
        p := unsafe.Pointer(&staticuint64s[*ptr])
        if goarch.BigEndian {
            p = add(p, 7)
        }
        return unsafe.String((*byte)(p), 1)
    }

    var p unsafe.Pointer
    if buf != nil && n <= len(buf) {
        p = unsafe.Pointer(buf)
    } else {
        p = mallocgc(uintptr(n), nil, false)
    }
    memmove(p, unsafe.Pointer(ptr), uintptr(n))
    return unsafe.String((*byte)(p), n)
}
```
