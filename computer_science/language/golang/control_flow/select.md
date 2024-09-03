# Select

Go 中的 `select` 可以用于 channel 的多路复用，例如在超时场景下，我们一方面需要等待操作的结果，另一方面需要限制操作的时间，超时直接结束，此时可以利用 `select` 同时监听多个 channel 的状态变化。

```go
select {
case res := <-ch:
    ...
case <-time.After(time.Second * 1):
    ...
}
```

`select` 本身的执行会阻塞当前 goroutine，直至某一个 `case` 满足条件并执行成功。

但是 `select` 支持 `default` 语句，如果此时所有 `case` 均不满足条件，无法完成写入或是读取操作，则会直接执行 `default` 的逻辑，从而避免阻塞。利用这一点，也可以实现非阻塞的写入或读取。

例如 `ch` 中缓冲区已满，如果此时要求程序的执行是非阻塞的，可以通过 `select` 和 `default` 关键字来实现。

```go
select {
case ch <- value:
    ...
default:
    ...
}
```

## 数据结构

对应于 `select` 中的 `case` 语句，每一个都是 `scase` 结构体，包含该条语句所引用的 channel 的结构体 `hchan`，以及发送或接收数据时所用到的元素 `elem`：

```go
// Select case descriptor.
type scase struct {
    c    *hchan         // chan
    elem unsafe.Pointer // data element
}
```

## 节点替换

在节点替换阶段，[walkStmt()](https://github.com/golang/go/blob/go1.22.0/src/cmd/compile/internal/walk/stmt.go#L15) 函数、[walkSelect](https://github.com/golang/go/blob/go1.22.0/src/cmd/compile/internal/walk/select.go#L15) 函数和 [walkSelectCases()](https://github.com/golang/go/blob/go1.22.0/src/cmd/compile/internal/walk/select.go#L33) 函数会对 `select` 中每个 `case` 分支进行代码逻辑优化和处理。

```go
func walkStmt(n ir.Node) ir.Node {
switch n.Op() {
    case ir.OSELECT:
        n := n.(*ir.SelectStmt)
        walkSelect(n)
        return n
    ...
    }
}

func walkSelect(sel *ir.SelectStmt) {
    ...
    init = append(init, walkSelectCases(sel.Cases)...)
    ...
}

func walkSelectCases(cases []*ir.CommClause) []ir.Node {
    ...
}
```

[walkSelectCases()](https://github.com/golang/go/blob/go1.22.0/src/cmd/compile/internal/walk/select.go#L33) 函数内部，根据 `case` 数量区分了 4 种场景，分别进行优化处理：

```go
func walkSelectCases(cases []*ir.CommClause) []ir.Node {
    ncas := len(cases)
    ...
    if ncas == 0 {
        ...
        return
    }
    if ncas == 1 {
        ...
        return
    }
    ...
    if ncas == 2 && dflt != nil {
        ...
        return
    }
    ...
    return
}
```

### Zero-case Select

#### 语句转换

当 `case` 数量为 0 时，会优化为调用 [block()](https://github.com/golang/go/blob/go1.22.0/src/runtime/select.go#L102) 函数，永久阻塞当前 goroutine：

```go
func walkSelectCases(cases []*ir.CommClause) []ir.Node {
    ...
    // optimization: zero-case select
    if ncas == 0 {
        return []ir.Node{mkcallstmt("block")}
    }
    ...
}

func block() {
    gopark(nil, nil, waitReasonSelectNoCases, traceBlockForever, 1) // forever
}
```

#### 代码示例

- 原代码：

    ```go
    select { }
    ```

- 转化后的示例代码：

    ```go
    block()
    ```

### Single Op

#### 语句转换

当 `case` 数量为 1 时，会优化为直接执行相关的通信操作和对应的函数体，并在结束时插入一条 `break` 语句，用于跳出 `select` 结构。

```go
func walkSelectCases(cases []*ir.CommClause) []ir.Node {
    ...
    // optimization: one-case select: single op.
    if ncas == 1 {
        ...
        l := cas.Init()
        if cas.Comm != nil { // not default:
            n := cas.Comm
            l = append(l, ir.TakeInit(n)...)
            ...
            l = append(l, n)
        }

        l = append(l, cas.Body...)
        l = append(l, ir.NewBranchStmt(base.Pos, ir.OBREAK, nil))
        return l
    }
    ...
}
```

此外，还针对接收数据的语句做了额外处理，如果接收语句的两个返回元素均为空标识符，则直接将 n 同样设置为空标识符，否则设置为接收两个值的赋值操作：

```go
func walkSelectCases(cases []*ir.CommClause) []ir.Node {
    ...
    // optimization: one-case select: single op.
    if ncas == 1 {
        ...
        if cas.Comm != nil { // not default:
            ...
            switch n.Op() {
            case ir.OSELRECV2:
                r := n.(*ir.AssignListStmt)
                if ir.IsBlank(r.Lhs[0]) && ir.IsBlank(r.Lhs[1]) {
                    n = r.Rhs[0]
                    break
                }
                r.SetOp(ir.OAS2RECV)
            ...
            }
            ...
        }
        ...
    }
    ...
}
```

#### 代码示例

- 原代码：

    ```go
    select { 
    case ch <- value:
        body
    }
    ```

- 转化后的示例代码：

    ```go
    ch <- value
    body
    break
    ```

### Single Non-blocking Op

#### 语句转换

对于包含两条 `case` 语句，且其中一条是 `default` 语句的情况，会将其转化为 `if` 语句，并调用 channel 中发送数据和接收数据的特定函数，实现非阻塞调用，并在最后添加 `break` 语句，用于跳出 `select` 结构。

```go
func walkSelectCases(cases []*ir.CommClause) []ir.Node {
    ...
    // optimization: two-case select but one is default: single non-blocking op.
    if ncas == 2 && dflt != nil {
        ...
        n := cas.Comm
        ir.SetPos(n)
        r := ir.NewIfStmt(base.Pos, nil, nil, nil)
        ...
        return []ir.Node{r, ir.NewBranchStmt(base.Pos, ir.OBREAK, nil)}
    }
    ...
}
```

对于新构建的 `if` 语句来说：

- `if` 语句的 `Init` 块，对应了 `case` 语句的 `Init` 块
- `if` 语句的 `Body` 块，对应了 `case` 语句中的 `Body` 块
- `if` 语句的 `Else` 块，对应了 `default` 语句中的 `Body` 块

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        if ncas == 2 && dflt != nil {
            ...
            r.SetInit(cas.Init())
            ...
            r.Body = cas.Body
            r.Else = append(dflt.Init(), dflt.Body...)
            ...
        }
        ...
    }
    ```

- `if` 语句的 `Cond` 块，根据具体的操作节点分别进行构造

  - 对应 `OSEND` 语句，`Cond` 块会直接转化为 [selectnbsend()](https://github.com/golang/go/blob/go1.22.0/src/runtime/chan.go#L693) 函数的调用，函数内部会非阻塞地调用 [chansend()](https://github.com/golang/go/blob/go1.22.0/src/runtime/chan.go#L160) 函数

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        if ncas == 2 && dflt != nil {
            ...
            var cond ir.Node
            switch n.Op() {
            case ir.OSEND:
                n := n.(*ir.SendStmt)
                ch := n.Chan
                cond = mkcall1(chanfn("selectnbsend", 2, ch.Type()), types.Types[types.TBOOL], r.PtrInit(), ch, n.Value)
            ...
            }

            r.Cond = typecheck.Expr(cond)
            ...
        }
        ...
    }

    func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
        return chansend(c, elem, false, getcallerpc())
    }
    ```

  - 对应 `OSELRECV2` 语句，`Cond` 块会被设置为一个临时变量
  - 之后会创建一个 [selectnbrecv()](https://github.com/golang/go/blob/go1.22.0/src/runtime/chan.go#L713) 函数的调用语句和赋值语句，并将其添加至 `Init` 块中
  - [selectnbrecv()](https://github.com/golang/go/blob/go1.22.0/src/runtime/chan.go#L713) 函数的第一个返回值会赋值给 `Cond` 块对应的临时变量

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        if ncas == 2 && dflt != nil {
            ...
            var cond ir.Node
            switch n.Op() {
            case ir.OSELRECV2:
                ...
                cond = typecheck.TempAt(base.Pos, ir.CurFunc, types.Types[types.TBOOL])
                fn := chanfn("selectnbrecv", 2, ch.Type())
                call := mkcall1(fn, fn.Type().ResultsTuple(), r.PtrInit(), elem, ch)
                as := ir.NewAssignListStmt(r.Pos(), ir.OAS2, []ir.Node{cond, n.Lhs[1]}, []ir.Node{call})
                r.PtrInit().Append(typecheck.Stmt(as))
            ...
            }

            r.Cond = typecheck.Expr(cond)
            ...
        }
        ...
    }

    func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected, received bool) {
        return chanrecv(c, elem, false)
    }
    ```

#### 代码示例

- 原代码：

    ```go
    select { 
    case ch <- value:
        body
    default:
        body2
    }

    select { 
    case value, ok <- ch:
        body
    default:
        body2
    }
    ```

- 转化后的示例代码：

    ```go
    if selectnbsend(ch, value) {
        body
    } else {
        body2
    }

    if selected, ok = selectnbrecv(&value, c); selected {
        body
    } else {
        body2
    }
    ```

### Multi Op

#### 语句转换

- 初始化

  - 更新计数器 `ncas`，排除 `default` 的场景，并初始化发送语句计数器 `nsends` 和发送语句计数器 `nrecvs`
  - 初始化 Multi Op 场景下的 `init` 节点，并最终返回该节点
  - 创建两个长度为 `ncas` 的数组，用于存储 `case` 语句和 `case` 语句在运行时的结构体，并将后者的初始化逻辑添加至 `init` 节点中
  - 创建一个长度为 `ncas` 二倍的数组，用于后续 [selectgo()](https://github.com/golang/go/blob/go1.22.0/src/runtime/select.go#L121C6-L121C14) 函数中排序使用

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        if dflt != nil {
            ncas--
        }
        casorder := make([]*ir.CommClause, ncas)
        nsends, nrecvs := 0, 0

        var init []ir.Node

        // generate sel-struct
        base.Pos = sellineno
        selv := typecheck.TempAt(base.Pos, ir.CurFunc, types.NewArray(scasetype(), int64(ncas)))
        init = append(init, typecheck.Stmt(ir.NewAssignStmt(base.Pos, selv, nil)))

        // No initialization for order; runtime.selectgo is responsible for that.
        order := typecheck.TempAt(base.Pos, ir.CurFunc, types.NewArray(types.Types[types.TUINT16], 2*int64(ncas)))
        ...
        return init
    }
    ```

- 注册 `case` 节点

  - 将 `case` 语句的初始化添加在 `init` 节点中
  - 过滤 `default` 语句

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        // register cases
        for _, cas := range cases {
            ir.SetPos(cas)

            init = append(init, ir.TakeInit(cas)...)

            n := cas.Comm
            if n == nil { // default:
                continue
            }
            ...
        }
        ...
    }
    ```

  - 根据 `OSEND` 和 `OSELRECV2` 节点，设置对应的索引 `i`，channel `c`，发送或接收的元素 `elem`，计数器 `sends` 或 `nrecvs`
  - 其中对应于索引，发送语句正排，接收语句倒排

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        // register cases
        for _, cas := range cases {
            ...
            var i int
            var c, elem ir.Node
            switch n.Op() {
            default:
                base.Fatalf("select %v", n.Op())
            case ir.OSEND:
                n := n.(*ir.SendStmt)
                i = nsends
                nsends++
                c = n.Chan
                elem = n.Value
            case ir.OSELRECV2:
                n := n.(*ir.AssignListStmt)
                nrecvs++
                i = ncas - nrecvs
                recv := n.Rhs[0].(*ir.UnaryExpr)
                c = recv.X
                elem = n.Lhs[0]
            }
            ...
        }
        ...
    }
    ```

  - 将 `case` 语句更新至 `casorder` 数组中索引所在的位置
  - 将 channel `c` 和数据元素 `elem` 更新至 `selv` 数组中索引所在位置的 `case` 结构体中

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        // register cases
        for _, cas := range cases {
            ...
            casorder[i] = cas

            setField := func(f string, val ir.Node) {
                r := ir.NewAssignStmt(base.Pos, ir.NewSelectorExpr(base.Pos, ir.ODOT, ir.NewIndexExpr(base.Pos, selv, ir.NewInt(base.Pos, int64(i))), typecheck.Lookup(f)), val)
                init = append(init, typecheck.Stmt(r))
            }

            c = typecheck.ConvNop(c, types.Types[types.TUNSAFEPTR])
            setField("c", c)
            if !ir.IsBlank(elem) {
                elem = typecheck.ConvNop(elem, types.Types[types.TUNSAFEPTR])
                setField("elem", elem)
            }
            ...
        }
        ...
    }
    ```

  - 校验计数器是否正确

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        if nsends+nrecvs != ncas {
            base.Fatalf("walkSelectCases: miscount: %v + %v != %v", nsends, nrecvs, ncas)
        }
        ...
    }
    ```

- 执行 [selectgo()](https://github.com/golang/go/blob/go1.22.0/src/runtime/select.go#L121C6-L121C14) 函数

  - 创建一条新的赋值语句，其中左值为临时变量 `chosen` 和 `recvOK`，右值为 [selectgo()](https://github.com/golang/go/blob/go1.22.0/src/runtime/select.go#L121C6-L121C14) 函数调用
    - `chosen` 用于接收最终被选中的 case 语句的索引
    - `recvOK` 用于表示接收操作是否成功
    - `fnInit` 用于存储调用函数前的初始化代码，编译器会做一些优化和 debug 功能
  - 将 `fnInit` 相关语句以及赋值语句 `r` 添加至 `select` 代码块中的 `init` 块中

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        // run the select
        base.Pos = sellineno
        chosen := typecheck.TempAt(base.Pos, ir.CurFunc, types.Types[types.TINT])
        recvOK := typecheck.TempAt(base.Pos, ir.CurFunc, types.Types[types.TBOOL])
        r := ir.NewAssignListStmt(base.Pos, ir.OAS2, nil, nil)
        r.Lhs = []ir.Node{chosen, recvOK}
        fn := typecheck.LookupRuntime("selectgo")
        var fnInit ir.Nodes
        r.Rhs = []ir.Node{mkcall1(fn, fn.Type().ResultsTuple(), &fnInit, bytePtrToIndex(selv, 0), bytePtrToIndex(order, 0), pc0, ir.NewInt(base.Pos, int64(nsends)), ir.NewInt(base.Pos, int64(nrecvs)), ir.NewBool(base.Pos, dflt == nil))}
        init = append(init, fnInit...)
        init = append(init, typecheck.Stmt(r))
        ...
    }
    ```

  - [selectgo()](https://github.com/golang/go/blob/go1.22.0/src/runtime/select.go#L121C6-L121C14) 函数内部会确认各 case 处理的优先级，以及通过循环，等待处理完成

    ```go
    // selectgo returns the index of the chosen scase, which matches the
    // ordinal position of its respective select{recv,send,default} call.
    // Also, if the chosen scase was a receive operation, it reports whether
    // a value was received.
    func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
        ...
    }
    ```

#### 代码示例

- 原代码：

    ```go
    select { 
    case ch <- value:
        body
    }
    ```

- 转化后的示例代码：

    ```go

    ```

## 参考

- <https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/>
