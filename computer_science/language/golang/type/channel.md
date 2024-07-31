# Channel

在 Go 的并发模型中，相较于使用共享内存，更推荐使用管道 `channel`，来进行通信。

在设计上，`channel` 的读取与发送遵循了先进先出的规则，内部通过互斥锁来实现并发控制。

## 数据结构

在运行时，`channel` 使用结构体 [hchan](https://github.com/golang/go/blob/go1.22.0/src/runtime/chan.go#L33) 来表示：

```go
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // size of the circular queue
    buf      unsafe.Pointer // points to an array of dataqsiz elements
    elemsize uint16
    closed   uint32
    elemtype *_type // element type
    sendx    uint   // send index
    recvx    uint   // receive index
    recvq    waitq  // list of recv waiters
    sendq    waitq  // list of send waiters

    lock mutex
}
```

- `qcount`、`dataqsiz`、`buf`、`sendx`、`recvx` 构成了一个循环队列，用于维护 `channel` 内部的缓冲区

- `elemsize` 和 `elemtype` 存储了 `channel` 所传递的元素的信息

- `sendq` 和 `recvq` 维护了目前被阻塞的 goroutine 列表，列表的数据结构为双向链表，[waitq](https://github.com/golang/go/blob/go1.22.0/src/runtime/chan.go#L54) 存储了链表的头尾节点，[sudog](https://github.com/golang/go/blob/master/src/runtime/runtime2.go#L356) 内部维护了 goroutine 相关信息，以及 `prev` 和 `next` 指针

    ```go
    type waitq struct {
        first *sudog
        last  *sudog
    }

    type sudog struct {
        g *g

        next *sudog
        prev *sudog
        ...
    }
    ```

- `lock` 为 `channel` 提供了并发控制

## 创建管道

### 使用实例

创建管道时，需要使用 `make` 关键字，并通过 `chan` 关键字加元素类型，共同指明 `channel` 的类型，同时 `channel` 也支持手动指定缓冲区的大小。

```go
ch1 := make(chan int)
ch2 := make(chan int, 10)
```

### 类型检查

编译器会在类型检查阶段，通过 [typecheck1()](https://github.com/golang/go/blob/go1.22.0/src/cmd/compile/internal/typecheck/typecheck.go#L218)函数和 [tcMake()](https://github.com/golang/go/blob/go1.22.0/src/cmd/compile/internal/typecheck/func.go#L514) 函数，区分 `make` 函数真正创建的类型，将 `OMAKE` 节点，转化为 `OMAKECHAN` 节点：

```go
func typecheck1(n ir.Node, top int) ir.Node {
    ...
    switch n.Op() {
    case ir.OMAKE:
        n := n.(*ir.CallExpr)
        return tcMake(n)
    ...
    }
}
```

- [tcMake()](https://github.com/golang/go/blob/go1.22.0/src/cmd/compile/internal/typecheck/func.go#L514) 函数首先获取第一个参数，判断当前所要创建的元素类型，并区分 `slice`、`map` 和 `channel` 执行不同的逻辑。

    ```go
    func tcMake(n *ir.CallExpr) ir.Node {
        args := n.Args
        ...
        l := args[0]
        l = typecheck(l, ctxType)
        t := l.Type()
        ...
        switch t.Kind() {
        case types.TSLICE:
            ...
        case types.TMAP:
            ...
        case types.TCHAN:
            ...
        ...
        }
        ...
    }
    ```

- 其次更新 `args` 切片的索引，设置为 `1`，当 `i < len(args)` 时，说明在调用 `make()` 函数时，还额外指定了 `size` 参数，对应于 `types.TCHAN` 类型，就是指缓冲区的大小，当未额外指定时，默认缓冲区大小为 `0`

    ```go
    func tcMake(n *ir.CallExpr) ir.Node {
        ...
        i := 1
        var nn ir.Node
        switch t.Kind() {
        case types.TCHAN:
            l = nil
            if i < len(args) {
                l = args[i]
                i++
                l = Expr(l)
                l = DefaultLit(l, types.Types[types.TINT])
                ...
            } else {
                l = ir.NewInt(base.Pos, 0)
            }
            ...
        ...
        }
        ...
    }
    ```

- 最终构造出新的 `OMAKECHAN` 节点并返回，实现 `OMAKE` 节点的转换逻辑
  
    ```go
    func tcMake(n *ir.CallExpr) ir.Node {
        ...
        var nn ir.Node
        switch t.Kind() {
        case types.TCHAN:
            ...
            nn = ir.NewMakeExpr(n.Pos(), ir.OMAKECHAN, l, nil)
        ...
        }
        ...
        return nn
    }
    ```

### 节点替换
