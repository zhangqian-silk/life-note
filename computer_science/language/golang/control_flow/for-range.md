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

对于最经典的 for 循环来说，对应于 AST 中的 `OFOR` 节点，在生成中间代码时，会构建为一个 [ForStmt](https://github.com/golang/go/blob/3959d54c0bd5c92fe0a5e33fedb0595723efc23b/src/cmd/compile/internal/ir/stmt.go#L218) 结构体

```go
// A ForStmt is a non-range for loop: for Init; Cond; Post { Body }
type ForStmt struct {
    miniStmt
    Label        *types.Sym
    Cond         Node
    Post         Node
    Body         Nodes
    DistinctVars bool
}

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

        // generate body
        s.startBlock(bBody)
        s.stmtList(n.Body)

        // tear down continue/break
        s.continueTo = prevContinue
        s.breakTo = prevBreak
        if lab != nil {
            lab.continueTarget = nil
            lab.breakTarget = nil
        }

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

        s.startBlock(bEnd)
    ...
    }
}
```
