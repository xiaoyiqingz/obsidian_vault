[[Go-channel内存泄漏问题]]

```go
func main() {
	c := make(chan int)
	
	go func() {
		n <- c  // 因为各种原因，此处channel在等待接收而一直阻塞，导致该gorountie一直无法退出，本质是该gorountie泄漏
		return
	}()

	return
}
```

而如果在 `main` 函数退出前，`channel c`  接收到数据，`goroutine` 可以在 `main`  前退出，即使不 `close channel` 也不会有内存泄漏，channel会被自动回收