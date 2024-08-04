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

```go
func walkSelectCases(cases []*ir.CommClause) []ir.Node {
    ...

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
