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
