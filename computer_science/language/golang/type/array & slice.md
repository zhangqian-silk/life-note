# 数组和切片

## 数组

数组是一种常见的容器类型，由一组相同类型的元素构成。系统会为数组分配一块连续的内存，各元素有序排列，可直接通过索引访问特定元素。

数组类型由元素类型加容器大小共同确定，所以在编译期，就必须确定具体的元素类型和容器大小，前者仍然支持 `any` 等接口，后者则必须使用常量来进行声明。编译期数据结构 [Array](https://github.com/golang/go/blob/07fc59199b9522bfe0d14f35c4391394efc336c9/src/cmd/compile/internal/types/type.go#L424) 如下所示：

```go
type Array struct {
    Elem  *Type // element type
    Bound int64 // number of elements; <0 if unknown yet
}
```

当数组的元素类型不同或是长度不同，均被认为是不同类型：

```go
var a1 [1]int
var a2 [2]int
fmt.Println(reflect.TypeOf(a1)) // "[1]int"
fmt.Println(reflect.TypeOf(a2)) // "[2]int"
fmt.Println(reflect.TypeOf(a1) == reflect.TypeOf(a2)) // "false"
```

当需要使用二维或多维数组时，例如矩阵，可通过数组嵌套的方式来进行实现：

```go
var a [2][2]int = [2][2]int{{0, 1}, {2, 3}}
fmt.Println("a[0][1]: ", a[0][1]) // "a[0][1]: 1"
fmt.Println("a[1][0]: ", a[1][0]) // "a[1][0]: 2"
```

### 初始化

默认情况下，数组中的每个元素会被初始化为对应的零值，整数类型为 0，指针类型为 nil，等等。同时，我们也可以使用字面量来初始化数组：

```go
a := [3]int{1, 2}
fmt.Println(a) // "[1 2 0]"
```

在初始化时，也可以使用省略号 `...` 让编译器来推导数组长度：

```go
a := [...]int{1, 2}
fmt.Println(a) // "[1 2]"
```

数组默认会按照字面量中的元素顺序进行赋值，必要的情况下，我们也可以手动指定元素及其索引：

```go
a1 := [5]int{1, 2: 3, 4}
fmt.Println(a1) // "[1 0 3 4 0]"
a2 := [...]int{2: 3, 4}
fmt.Println(a2) // "[0 0 3 4]"
```

所有未指定索引的元素，会默认排列在前一个元素后面，例如元素 `3` 的索引手动指定为 `2`，此时紧跟着的元素 `4` 的索引默认为 `3`。当然，第一个元素若没有指定索引，则默认为索引 `0`。而 `...` 在这种场景下，也同样适用，会自动推导数组长度，对未指定的元素赋值为零值。

### 访问

数组中的元素可以通过下标进行访问，内置的 `len` 函数将返回数组中的元素个数，通过下标 `n` 访问数组 `a` 时，必须注意索引范围，即 `a[n]` 必须满足 $0 \leq n < len(array)$：

```go
a := [3]int{1, 2, 3}
fmt.Println(a[0])           // "1"
fmt.Println(a[len(a)-1])    // "3"
fmt.Println(a[-1])          // compile error
fmt.Println(a[len(a)])      // compile error
```

还支持通过 `for range` 进行遍历处理：

```go
a := [3]int{1, 2: 3}
// "0 1"
// "1 0"
// "2 3"
for index, value := range a {
    fmt.Println(index, value)
}
```

### 运算

当数组类型相同（数组长度一致且元素类型相同）且数组中的元素支持比较时，数组也可以通过 `==` 和 `!=` 进行比较，当且仅当数组中的所有元素均相等时，两个数组相等：

```go
a := [2]int{1, 2}
b := [...]int{1, 2}
c := [2]int{1, 3}
fmt.Println(a == b, a == c, b == c) // "true false false"

d := [2]int64{1,2}
fmt.Println(a == d) // compile error
```
