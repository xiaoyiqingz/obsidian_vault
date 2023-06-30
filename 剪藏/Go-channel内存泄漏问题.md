一说到 go channel，很多人会使用“优秀”“哲学”这些词汇来描述。殊不知，go channel 恐怕还是 golang 中最容易造成问题的特性之一。很多情况下，我们使用 go channel 时，常常以为可以关闭 channel，但实际上却没有关闭，这就是导致 go channel 内存泄漏的元凶。

> 阅读本文前要求读者熟悉 go channel 的基本知识。如果你不够了解 go channel，那么可以先阅读[《新手使用 go channel 需要注意的问题》](https://juejin.cn/post/7031004528929406984 "https://juejin.cn/post/7031004528929406984")。本文会默认你已经了解相关内容。

## 情境一：`select-case` 误用导致的内存泄露

废话说少，先看代码。

```go
func TestLeakOfMemory(t *testing.T) {
	fmt.Println("NumGoroutine:", runtime.NumGoroutine())
	chanLeakOfMemory()
	time.Sleep(time.Second * 3) // 等待 goroutine 执行，防止过早输出结果
	fmt.Println("NumGoroutine:", runtime.NumGoroutine())
}

func chanLeakOfMemory() {
	errCh := make(chan error) // (1)    
	go func() { // (5)
	    time.Sleep(2 * time.Second)
	    errCh <- errors.New("chan error") // (2)
	    fmt.Println("finish sending")
	}()
	var err error
	select {
		case <-time.After(time.Second): // (3) 大家也经常在这里使用 <-ctx.Done()
			fmt.Println("超时")
		case err = <-errCh: // (4)
		    if err != nil {
			    fmt.Println(err)
			} else {
				 fmt.Println(nil)
			}    
	} 
}
```


大家认为输出的结果是什么？正确的输出结果如下：

```bash
NumGoroutine: 2
超时
NumGoroutine: 3
```

这是 `go channel` 导致内存泄漏的经典场景。根据输出结果（开始有两个  `goroutine` ，结束时有三个  `goroutine` ），我们可以知道，直到测试函数结束前，仍有一个  `goroutine`  没有退出。原因是由于 (1) 处创建的 errCh 是不含缓存队列的 channel，如果 channel 只有发送方发送，那么发送方会阻塞；如果 channel 只有接收方，那么接收方会阻塞。

我们可以看到由于没有发送方往 errCh 发送数据，所以 (4) 处代码一直阻塞。直到 (3) 处超时后，打印“超时”，函数退出，(4) 处代码都未接收成功。而 (2) 处的所在的 goroutine 在“超时”被打印后，才开始发送。由于外部的 goroutine 已经退出了，errCh 没有接收者，导致 (2) 处一直阻塞。因此 (2) 处代码所在的协程一直未退出，造成了内存泄漏。如果代码中有许多类似的代码，或在 for 循环中使用了上述形式的代码，随着时间的增长会造成多个未退出的 gorouting，最终导致程序 OOM。

这种情况其实还比较简单。我们只需要为 channel 增加一个缓存队列。即把 (1) 处代码改为 *`errCh := make(chan error, 1)`*  即可。修改后输出如下所示，可知我们创建的 goroutine 已经退出了。

```bash
NumGoroutine: 2
超时
NumGoroutine: 2
```

可能会有人想要使用 `defer close(errCh)` 关闭 channel。比如把 (1) 处代码改为如下形式(错误)：

```go
errCh := make(chan error)
defer close(errCh)
```

由于 (2) 处代码没有接收者，所以一直阻塞。直到 `close(errCh)` 运行，(2) 处仍在阻塞。这导致关闭 channel 时，仍有 goroutine 在向 errCh 发送。然而在 golang 中，在向 channel 发送时不能关闭 channel，否则会 panic。因此这种方式是错误的。

又或在 (5) 处 goroutine 的第一句加上 `defer close(errCh)`。由于 (2) 处阻塞， `defer close(errCh)` 会一直得不到执行。因此也是错误的。 即便对调 (2) 处和 (4) 处的发送者和接收者，也会因为 channel 关闭，导致输出无意义的零值。

## 情景二：`for-range` 误用导致的内存泄露

上述示例中只有一个发送者，且只发送一次，所以增加一个缓存队列即可。但在其他情况下，可能不止有一个发送者（或者不只发送一次），所以这个方案要求，缓存队列的容量需要和发送次数一致。一旦缓存队列容量被用完后，再有发送者发送就会阻塞发送者 goroutine。如果恰好此时接收者退出了，那么仍然至少会有一个 goroutine 无法退出，从而造成内存泄漏。就比如下面的代码。不知道经过上面的讲解，读者是否能够发现其中的问题。


```go
func TestLeakOfMemory2(t *testing.T) {
   fmt.Println("NumGoroutine:", runtime.NumGoroutine())
   chanLeakOfMemory2()
   time.Sleep(time.Second * 3) // 等待 goroutine 执行，防止过早输出结果
   fmt.Println("NumGoroutine:", runtime.NumGoroutine())
}

func chanLeakOfMemory2() {
   ich := make(chan int, 100) // (3)
   // sender
   go func() {
      defer close(ich)
      for i := 0; i < 10000; i++ {
         ich <- i
         time.Sleep(time.Millisecond) // 控制一下，别发太快
      }
   }()
   // receiver
   go func() {
      ctx, cancel := context.WithTimeout(context.Background(), time.Second)
      defer cancel()
      for i := range ich { // (2)
         if ctx.Err() != nil { // (1)
            fmt.Println(ctx.Err())
            return
         }
         fmt.Println(i)
      }
   }()
}

// Output:
// NumGoroutine: 2
// 0
// 1
// ...(省略)...
// 789
// context deadline exceeded
// NumGoroutine: 3

```

我们聪明地使用了 channel 的缓存队列。我们以为我们循环发送，发完之后就会把 channel 关闭。而且我们使用 for range 获取 channel 的值，会一直获取，直到 channel 关闭。但在代码 (1) 处，接收者的 goroutine 中，我们加了一个判断语句。这会让代码 (2) 处的 channel 还没被接收完就退出了接收者 goroutine。尽管代码 (3) 处有缓存，但是因为发送 channel 在 for 循环中，缓存队列很快就会被占满，阻塞在第 101 的位置。所以这种情况我们要使用一个额外的 stop channel 来终结发送者所在的 goroutine。方式如下：

```go
func TestLeakOfMemory2(t *testing.T) {
   fmt.Println("NumGoroutine:", runtime.NumGoroutine())
   chanLeakOfMemory2()
   time.Sleep(time.Second * 3) // 等待 goroutine 执行，防止过早输出结果
   fmt.Println("NumGoroutine:", runtime.NumGoroutine())
}

func chanLeakOfMemory2() {
   ich := make(chan int, 100)
   stopCh := make(chan struct{})
   // sender
   go func() {
      defer close(ich)
      for i := 0; i < 10000; i++ {
         select {
         case <-stopCh:
             return
         case ich <- i:
         }
         time.Sleep(time.Millisecond) // 控制一下，别发太快
      }
   }()
   // receiver
   go func() {
      ctx, cancel := context.WithTimeout(context.Background(), time.Second)
      defer cancel()
      for i := range ich {
         if ctx.Err() != nil {
            fmt.Println(ctx.Err())
            close(stopCh)
            return
         }
         fmt.Println(i)
      }
   }()
}

// Output:
// NumGoroutine: 2
// 0
// 1
// ...(省略)...
// 789
// context deadline exceeded
// NumGoroutine: 2

```

可能有人会问，要是接收者 goroutine 关闭 stop channel 的时候，发送者又继续发送了怎么办？不会内存泄漏吗？

答案是不会的。因为只可能存在两种情况，一种是发送者把数据发送到了缓存中，发送者想要继续发送时，select 发现 stop channel 已经关闭，发送者 goroutine 会退出；一种是 channel 没有缓存了，发送者只能阻塞，此时 select 发现 stop channel 已经关闭，发送者 goroutine 也会退出。

总之，通常情况下，我们只会遇到这两种 go channel 造成内存泄漏的情况（一个发送者导致的内存泄漏和多个发送者导致的内存泄漏）。如果你了解其他 go channel 造成的内存泄漏情况，也欢迎在评论区留言。

让我们仔细观察上述两个内存泄漏的案例：

```go
func chanLeakOfMemory() {
   errCh := make(chan error) // (1)
   go func() chan error { // (5)
      time.Sleep(2 * time.Second)
      errCh <- errors.New("chan error") // (2)
      return errCh
   }()

   var err error
   select {
   case <-time.After(time.Second): // (3) 大家也经常在这里使用 <-ctx.Done()
      fmt.Println("超时")
   case err = <-errCh: // (4)
      if err != nil {
         fmt.Println(err)
      } else {
         fmt.Println(nil)
      }
   }
}

func chanLeakOfMemory2() {
   ich := make(chan int, 100) // (3)
   // sender
   go func() {
      defer close(ich)
      for i := 0; i < 10000; i++ {
         ich <- i
         time.Sleep(time.Millisecond) // 控制一下，别发太快
      }
   }()
   // receiver
   go func() {
      ctx, cancel := context.WithTimeout(context.Background(), time.Second)
      defer cancel()
      for i := range ich { // (2)
         if ctx.Err() != nil { // (1)
            fmt.Println(ctx.Err())
            return
         }
         fmt.Println(i)
      }
   }()
}

```

可以发现：

不论发送者发送一次还是多次，如果接收者所在 goroutine 能够在接收完 channel 中的数据之后结束，那么就不会造成内存泄漏；或者说接收者能够在发送者停止发送后再结束，就不会造成内存泄露。

如果接收者需要在 channel 关闭之前提前退出，为防止内存泄漏，在发送者与接收者发送次数是一对一时，应设置 channel 缓冲队列为 1；在发送者与接收者的发送次数是多对多时，应使用专门的 stop channel 通知发送者关闭相应 channel。

## 参考文章

*   [如何退出协程 goroutine (超时场景)](https://link.juejin.cn/?target=https%3A%2F%2Fgeektutu.com%2Fpost%2Fhpg-timeout-goroutine.html "https://geektutu.com/post/hpg-timeout-goroutine.html")
*   [go channel 关闭的那些事儿](https://juejin.cn/post/7033671944587182087 "https://juejin.cn/post/7033671944587182087")
