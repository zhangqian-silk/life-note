# Sync

## Mutex

> [wiki/互斥锁](https://zh.wikipedia.org/wiki/%E4%BA%92%E6%96%A5%E9%94%81)

互斥锁（英语：Mutual exclusion，缩写 Mutex）是一种用于多线程编程中，防止两条线程同时对同一公共资源（比如全局变量）进行读写的机制。

### 数据结构

Golang 中互斥锁的数据结构为 [sync.Mutex](https://github.com/golang/go/blob/go1.22.0/src/sync/mutex.go#L34)，其中 `state` 用来表示互斥锁当前的状态，`sema` 是用于控制锁的信号量。

```go
type Mutex struct {
    state int32
    sema  uint32
}
```

针对于 `state` 字段，其中低三位用于表示状态，其他位用于表示当前正在等待的 goroutine 的数量：

- `mutexLocked`：互斥锁是否锁定
- `mutexWoken`：表示从正常模式被唤醒
- `mutexStarving`：从当前互斥锁进入饥饿状态
- `m.state >> mutexWaiterShift`：当前正在等待的 goroutine 的数量

```go
mutexLocked = 1 << iota // mutex is locked
mutexWoken
mutexStarving
mutexWaiterShift = iota
```

### 公平性

> [wiki/Starvation_(computer_science)](https://en.wikipedia.org/wiki/Starvation_(computer_science))

饥饿是指在并发环境下，某一个进程永远无法获取其运行所需资源的现象，在调度算法或互斥锁算法的场景下，都有可能出现。

在 Golang 中，互斥锁存在两种模式，正常模式(normal)和饥饿模式(starvation)，在普通模式下，互斥锁是一种非公平锁，goroutine 在申请锁时，会直接去竞争锁，如果获取到，就会直接占有锁，执行后续逻辑，获取不到才会进入等待队列，队列中会按照先进先出的顺序来获取锁。

从锁的分配的角度来说，如果等待队列非空，且同时有一个新的 goroutine 在获取锁，那么在竞争时，后者会极大概率胜出，因为他本身就持有 CPU 资源，可以减少唤醒 goroutine 的开销。

但是相对的，如果竞争锁的 goroutine 特别多，每次锁均分配给了新参与竞价的 goroutine，那么队列中末尾的 goroutine，很有可能永远获取不到锁，也就是出现饥饿现象。

在饥饿模式下，互斥锁是一种公平锁，锁的所有权会直接移交给队列中的首位 goroutine，确保队列中的所有 goroutine 均有机会分配资源来执行，所有想要获取锁的 goroutine 直接进入等待队列的尾部。

总的来说，正常模式，即非公平锁，会减少整体的性能开销，但是会出现饥饿现象，饥饿模式，即公平锁，能够确保队列中的所有 goroutine 均会获取锁，避免尾部的非预期内的延迟，亦即饥饿现象，但是会有较多的资源浪费在唤醒等待中的 goroutine。

在目前的设计上，如果 goroutine 的等待时间超过了 1ms，即 1e6 ns，会将互斥锁切换至饥饿模式。相对应的，如果 goroutine 获取到了锁，等待时间小于 1ms，或是位于等待队列尾部，会将互斥锁切换至正常模式。

```go
starvationThresholdNs = 1e6
```

## 引用

- <https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/>
