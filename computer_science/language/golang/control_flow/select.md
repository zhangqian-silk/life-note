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
  - 初始化 Multi Op 场景下的 `init` 节点切片，并最终返回该切片
  - 创建长度为 `ncas` 的切片 `casorder`，用于存储 `case` 语句
  - 创建长度为 `ncas` 的数组 `selv`，用于存储 `case` 语句在运行时的结构体，并将其的初始化逻辑添加至 `init` 节点中
  - 创建一个长度为 `ncas` 二倍的数组，用于后续 [selectgo()](https://github.com/golang/go/blob/go1.22.0/src/runtime/select.go#L121C6-L121C14) 函数中排序使用（用来存储轮询顺序和锁定顺序）

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

- 执行 [selectgo()](https://github.com/golang/go/blob/go1.22.0/src/runtime/select.go#L121) 函数，用于确认最终选中的 case 语句

  - 创建一条新的赋值语句，其中左值为临时变量 `chosen` 和 `recvOK`，右值为 [selectgo()](https://github.com/golang/go/blob/go1.22.0/src/runtime/select.go#L121) 函数调用
    - `chosen` 用于接收最终被选中的 case 语句的索引
    - `recvOK` 用于表示接收操作是否成功
    - `fnInit` 用于存储调用函数前的初始化代码，编译器会做一些优化和 debug 功能
  - 将 `fnInit` 相关语句以及赋值语句 `r` 添加至 `init` 列表中

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

  - [selectgo()](https://github.com/golang/go/blob/go1.22.0/src/runtime/select.go#L121) 函数内部会确认各 case 处理的优先级，以及通过循环，等待处理完成

    ```go
    // selectgo returns the index of the chosen scase, which matches the
    // ordinal position of its respective select{recv,send,default} call.
    // Also, if the chosen scase was a receive operation, it reports whether
    // a value was received.
    func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
        ...
    }
    ```

- 定义分发函数 `dispatch`

  - 定义节点数组 `list`，用于存储 case 语句最终执行的代码块
  - 如果 case 语句是 `OSELRECV2` 操作节点，则将其转为赋值语句，如果第二个返回值变量不为空，则将 `recvOK` 的值赋值给 `n.Lhs[1]`
  - 将 case 语句中的 `body` 代码块都添加至 list 中
  - 额外向 list 中添加一条 `break` 语句

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        // dispatch cases
        dispatch := func(cond ir.Node, cas *ir.CommClause) {
            var list ir.Nodes

            if n := cas.Comm; n != nil && n.Op() == ir.OSELRECV2 {
                n := n.(*ir.AssignListStmt)
                if !ir.IsBlank(n.Lhs[1]) {
                    x := ir.NewAssignStmt(base.Pos, n.Lhs[1], recvOK)
                    list.Append(typecheck.Stmt(x))
                }
            }

            list.Append(cas.Body.Take()...)
            list.Append(ir.NewBranchStmt(base.Pos, ir.OBREAK, nil))
            ...
        }
        ...
    }
    ```

  - 构建最终执行的节点 `r`
    - 如果 `cond` 代码块存在，则创建一个条件语句，满足条件时执行 `list` 代码块中逻辑
    - 如果 `cond` 代码块不存在，则直接创建一个代码块语句，并执行 `list` 代码块中逻辑
  - 将 `r` 添加至 `init` 列表中

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        // dispatch cases
        dispatch := func(cond ir.Node, cas *ir.CommClause) {
            ...
            var r ir.Node
            if cond != nil {
                cond = typecheck.Expr(cond)
                cond = typecheck.DefaultLit(cond, nil)
                r = ir.NewIfStmt(base.Pos, cond, list, nil)
            } else {
                r = ir.NewBlockStmt(base.Pos, list)
            }

            init = append(init, r)
        }
        ...
    }
    ```

- 通过分化函数，转化所有 case 语句与 default 语句

  - 如果存在 default 语句，则进行转化，其中 `cond` 对应的逻辑为 `chosen < 0`

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        if dflt != nil {
            ir.SetPos(dflt)
            dispatch(ir.NewBinaryExpr(base.Pos, ir.OLT, chosen, ir.NewInt(base.Pos, 0)), dflt)
        }
        ...
    }
    ```

  - 遍历转化 case 语句，其中 `cond` 对应的逻辑为 `chosen == i`
  - 如果 i 为最后一个索引，即 `len(casorder)-1`，则不指定 `cond` 代码块

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        for i, cas := range casorder {
            ir.SetPos(cas)
            if i == len(casorder)-1 {
                dispatch(nil, cas)
                break
            }
            dispatch(ir.NewBinaryExpr(base.Pos, ir.OEQ, chosen, ir.NewInt(base.Pos, int64(i))), cas)
        }
        ...
    }
    ```

- 返回转化结果

  - 上述所有逻辑处理完成后，返回最终转化的节点列表

    ```go
    func walkSelectCases(cases []*ir.CommClause) []ir.Node {
        ...
        return init
    }
    ```

#### 代码示例

- 原代码：

    ```go
    select { 
    case ch <- value1:
        body1
    case value2, ok <- ch2:
        body2
    case value3, _ <- ch3:
        body3
    default:
        body4
    }
    ```

- 转化后的示例代码：

  - 其中 case 语句在 `selv` 切片中对应的索引值，发送语句正排，接收语句倒排
  - 发送和接收操作，全部在 `selectgo()` 函数中进行处理

    ```go
    selv := [3]scase{}
    order := [6]uint16
    for i, cas := range cases {
        c := scase{}
        c.kind = ...
        c.elem = ...
        c.c = ...
    }
    chosen, revcOK := selectgo(selv, order, 3)
    if chosen < 0 {
        body4
        break
    }
    if chosen == 0 {
        ...
        body1
        break
    }
    if chosen == 1 {
        ...
        body3
        break
    }
    if chosen == 2 {
        ...
        ok = revcOK
        body2
        break
    }
    ```

### [selectgo()](https://github.com/golang/go/blob/go1.22.0/src/runtime/select.go#L121)

- 初始化

  - 限制 case 的最大数量为 `1<<16`，即 65535
  - 声明一些重要变量
    - `ncases`：case 总数
    - `scases`：case 的切片
    - `pollorder`：channel 的轮询顺序
    - `lockorder`：channel 的锁定顺序

    ```go
    func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
        // NOTE: In order to maintain a lean stack size, the number of scases
        // is capped at 65536.
        cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
        order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

        ncases := nsends + nrecvs
        scases := cas1[:ncases:ncases]
        pollorder := order1[:ncases:ncases]
        lockorder := order1[ncases:][:ncases:ncases]
        // NOTE: pollorder/lockorder's underlying array was not zero-initialized by compiler.
    
        ...
    }
    ```

  - 随机交换 `pollorder` 中的元素，生成随机的轮询顺序
    - 同时优化下 case 的数量，即 `norder`，排除不存在 `channel` 的 case
    - 交换时，通过 [cheaprandn()](https://github.com/golang/go/blob/go1.22.0/src/runtime/rand.go#L222) 函数随机生成一个范围为 $[0, norder+1)$ 的整数

    ```go
    func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
        ...
        // generate permuted order
        norder := 0
        for i := range scases {
            cas := &scases[i]

            // Omit cases without channels from the poll and lock orders.
            if cas.c == nil {
                cas.elem = nil // allow GC
                continue
            }

            j := cheaprandn(uint32(norder + 1))
            pollorder[norder] = pollorder[j]
            pollorder[j] = uint16(i)
            norder++
        }
        pollorder = pollorder[:norder]
        lockorder = lockorder[:norder]
        ...
    }
    ```

  - 将 `lockorder` 构建为一个最大堆
    - 各元素间通过 `c.sortkey()`，即 channel 对应的地址进行比较
    - 在外层循环中，每次循环结束，`lockorder` 中的前 `i` 个元素会被调整为一个极大堆
    - 在内层循环中，每次循环会判断当前元素 `j` 的父节点，即 `(j-1)/2`，与 `i` 所对应的元素的大小，若 `j` 小于 `i`，则交换元素位置，并继续向上寻找 `j` 的父元素做比较，直至找到大于 `i` 的元素或找到堆顶元素
      - 内层循环中，每次比较理论上应该交换 `j` 节点与其父节点的值，将 `i` 节点的值一路交换上去
      - 但是 `j` 的父节点再下次循环中，仍然可能和下一个父节点中的值做交换，所以每次循环中仅将 `j` 的父节点的值赋给 `j` 节点，在循环结束后，再给 `j` 节点赋值正确的值，即 `i` 节点的值

    ```go
    func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
        ...
        // sort the cases by Hchan address to get the locking order.
        // simple heap sort, to guarantee n log n time and constant stack footprint.
        for i := range lockorder {
            j := i
            // Start with the pollorder to permute cases on the same channel.
            c := scases[pollorder[i]].c
            for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
                k := (j - 1) / 2
                lockorder[j] = lockorder[k]
                j = k
            }
            lockorder[j] = pollorder[i]
        }
        ...
    }
    ```

  - 实现堆排序，将 `lockorder` 中的元素按照 channel 的地址升序进行排列
    > 每次循环将堆顶与堆在数组中的末尾元素交换，使得数组末端有序
    > 再将非有序的部分重新调整结构，使其满足最大堆的形式
    > 重复以上流程直至最终数组有序
    - 每次先将 `i` 的值，更新为索引为 `0` 的值，即当前堆中的最大值
    - 通过循环，将切片中 $[1, i]$ 部分的元素，在 $[0, i-1]$ 范围内重新排序为最大堆
      - 先将 `k` 更新为 `j` 节点的左子树，如果此时 `k` 的右侧元素已经为排序后的部分，则结束循环
      - 如果此时 `k+1` 即右子树的值更大且不属于有序的部分，则通过 `k++` 将 `k` 更新为 `j` 的右子树
      - 如果此时 `k` 的值大于 `i` 的值，则交换 `j` 与 `k` 的值，将 `j` 更新为子节点 `k`，并继续循环
      - 出于同样的原因，在交换 `j` 与 `k` 的值时，仅仅将 `k` 的值赋给 `j`，在最终循环结束后，将 `i` 的值赋值给 `j`，完成数据交换

    ```go
    func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
        ...
        for i := len(lockorder) - 1; i >= 0; i-- {
            o := lockorder[i]
            c := scases[o].c
            lockorder[i] = lockorder[0]
            j := 0
            for {
                k := j*2 + 1
                if k >= i {
                    break
                }
                if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
                    k++
                }
                if c.sortkey() < scases[lockorder[k]].c.sortkey() {
                    lockorder[j] = lockorder[k]
                    j = k
                    continue
                }
                break
            }
            lockorder[j] = o
        }
        ...
    }
    ```

  - 通过 [sellock()](https://github.com/golang/go/blob/go1.22.0/src/runtime/select.go#L33) 函数，按照上述确定的 `lockorder` 中的顺序，对 `scases` 中的 case 语句进行加锁处理
    - `lockorder` 按照 channel 的地址有序排列后，在加锁时可以跳过相同的实例，避免重复加锁导致死锁

    ```go
    func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
        ...
        // lock all the channels involved in the select
        sellock(scases, lockorder)
        ...
    }

    func sellock(scases []scase, lockorder []uint16) {
        var c *hchan
        for _, o := range lockorder {
            c0 := scases[o].c
            if c0 != c {
                c = c0
                lock(&c.lock)
            }
        }
    }
    ```

## 常见问题

## 参考

- <https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/>
