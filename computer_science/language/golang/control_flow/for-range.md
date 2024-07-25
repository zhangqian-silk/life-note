# For-Range

针对于容器类型，即数组、切片、哈希表，可以通过 `for-range` 来遍历集合中所有元素，来替换传统的 for 循环，在使用上也更为简洁。

- 切片：

```go
slice := []int{1, 2, 3, 4, 5}
for index, value := range slice {
    fmt.Printf("索引: %d, 值: %d\n", index, value)
}
```

- 哈希表：

```go
myMap := map[string]int{
    "a": 10,
    "b": 20,
    "c": 30,
}
for key, value := range myMap {
    fmt.Printf("键: %s, 值: %d\n", key, value)
}
```

## For 循环

对于最经典的 `for` 循环来说，在编译器中会构建为一个 [ForStmt](https://github.com/golang/go/blob/go1.22.0/src/cmd/compile/internal/ir/stmt.go#L218) 结构体，其中 `init`、`Cond`、`Post`、`Body` 代表了 `for` 循环必备的四块代码块。

```go
type miniStmt struct {
    miniNode
    init Nodes
}

// A ForStmt is a non-range for loop: for Init; Cond; Post { Body }
type ForStmt struct {
    miniStmt
    Label        *types.Sym
    Cond         Node
    Post         Node
    Body         Nodes
    DistinctVars bool
}
```

在生成 SSA 代码时，即 [stmt()](https://github.com/golang/go/blob/go1.22.0/src/cmd/compile/internal/ssagen/ssa.go#L1431) 方法中，会真正构建 `for` 循环的执行逻辑：

- 针对 `OFOR` 节点，创建 `Cond`、`Body`、`Incr`、`End` 四个代码块
  - 四个代码块分别对应了 `for` 循环的特定逻辑，即 `for Ninit; Cond; Incr { Body }`

    ```go
    func (s *state) stmt(n ir.Node) {
        ...
        switch n.Op() {
        case ir.OFOR:
            // OFOR: for Ninit; Left; Right { Nbody }
            // cond (Left); body (Nbody); incr (Right)
            n := n.(*ir.ForStmt)
            base.Assert(!n.DistinctVars) // Should all be rewritten before escape analysis
            bCond := s.f.NewBlock(ssa.BlockPlain)
            bBody := s.f.NewBlock(ssa.BlockPlain)
            bIncr := s.f.NewBlock(ssa.BlockPlain)
            bEnd := s.f.NewBlock(ssa.BlockPlain)

            // ensure empty for loops have correct position; issue #30167
            bBody.Pos = n.Pos()
            ...
        ...
        }
    }
    ```

- 构建 `Cond` 代码块，并从当前的结束代码块跳转至 `Cond` 代码块
  - 如果 `Cond` 代码块存在，则根据其结果为 `true` 还是 `false`，分别跳转至 `Body` 和 `End` 代码块
  - 如果 `Cond` 代码块为空，说明循环的条件表达式始终为真，则跳转至 `Body` 代码块

    ```go
    func (s *state) stmt(n ir.Node) {
        ...
        switch n.Op() {
        case ir.OFOR:
            ...
            // first, jump to condition test
            b := s.endBlock()
            b.AddEdgeTo(bCond)

            // generate code to test condition
            s.startBlock(bCond)
            if n.Cond != nil {
                s.condBranch(n.Cond, bBody, bEnd, 1)
            } else {
                b := s.endBlock()
                b.Kind = ssa.BlockPlain
                b.AddEdgeTo(bBody)
            }
            ...
        ...
        }
    }
    ```

- 设置 `continue` 和 `break` 的目标块，并处理标签相关逻辑
  - 当触发 `continue` 语句时，跳转至 `Incr` 代码块，开始下一次循环
  - 当触发 `break` 语句时，跳转至 `End` 代码块，结束当前循环

    ```go
    func (s *state) stmt(n ir.Node) {
        ...
        switch n.Op() {
        case ir.OFOR:
            ...
            // set up for continue/break in body
            prevContinue := s.continueTo
            prevBreak := s.breakTo
            s.continueTo = bIncr
            s.breakTo = bEnd
            var lab *ssaLabel
            if sym := n.Label; sym != nil {
                // labeled for loop
                lab = s.label(sym)
                lab.continueTarget = bIncr
                lab.breakTarget = bEnd
            }
            ...
        ...
        }
    }
    ```

- 构建 `Body` 代码块，处理循环体内部逻辑
  - 上述设置的 `continue` 和 `break` 的目标代码块，将针对 `Body` 代码块中的逻辑生效

    ```go
    func (s *state) stmt(n ir.Node) {
        ...
        switch n.Op() {
        case ir.OFOR:
            ...
            // generate body
            s.startBlock(bBody)
            s.stmtList(n.Body)
            ...
        ...
        }
    }
    ```

- 恢复 `continue` 和 `break` 的目标代码块
  - `Body` 代码块相关的逻辑已经构建结束，需要恢复原本的设置，例如循环嵌套的场景

    ```go
    func (s *state) stmt(n ir.Node) {
        ...
        switch n.Op() {
        case ir.OFOR:
            ...
            // tear down continue/break
            s.continueTo = prevContinue
            s.breakTo = prevBreak
            if lab != nil {
                lab.continueTarget = nil
                lab.breakTarget = nil
            }
            ...
        ...
        }
    }
    ```

- 构建 `Incr` 代码块
  - 设置 `Body` 代码块（如果存在）跳转至 `Incr` 代码块
  - 设置 `Incr` 代码块（如果存在）跳转至 `Cond` 代码块

    ```go
    func (s *state) stmt(n ir.Node) {
        ...
        switch n.Op() {
        case ir.OFOR:
            ...
            // done with body, goto incr
            if b := s.endBlock(); b != nil {
                b.AddEdgeTo(bIncr)
            }

            // generate incr
            s.startBlock(bIncr)
            if n.Post != nil {
                s.stmt(n.Post)
            }
            if b := s.endBlock(); b != nil {
                b.AddEdgeTo(bCond)
                // It can happen that bIncr ends in a block containing only VARKILL,
                // and that muddles the debugging experience.
                if b.Pos == src.NoXPos {
                    b.Pos = bCond.Pos
                }
            }
            ...
        ...
        }
    }
    ```

- 构建 `End` 代码块，需要注意的是 `End` 代码块仅能通过以下两种方式跳转
  - 通过 `Cond` 代码块在 `false` 的分支下跳转
  - 通过 `Body` 代码块在 `break` 语句下跳转

    ```go
    func (s *state) stmt(n ir.Node) {
        ...
        switch n.Op() {
        case ir.OFOR:
            ...
            s.startBlock(bEnd)
        ...
        }
    }
    ```
