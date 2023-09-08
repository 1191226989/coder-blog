---
title: Golang协程退出和泄漏问题
created: '2023-09-08T08:39:25.022Z'
modified: '2023-09-08T10:11:21.600Z'
---

# Golang协程退出和泄漏问题

1. main函数return退出，系统会强制关闭所有已启动的协程。其他非main函数return退出，其开启的子协程不会退出。例如：A不是main函数，A开启B协程，然后A执行完毕return，B协程不受影响也不会退出。

2. 所有协程在golang语言层面都是同一优先级别，不管子协程在哪个函数启动

3. 程序不是常驻内存，如果有协程泄露的情况，程序退出后系统也会自动回收运行时资源。但是如果程序代码是常驻内存服务执行，例如 http server，每接收到一个请求就启动一个新的协程，每次启动的 goroutine 如果都得不到有效释放，那么服务会造成资源耗尽崩溃。


#### 案例分析
##### 1. channel 发送不接收

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func gen(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		for _, n := range nums {
			out <- n
		}
		close(out)
	}()
	return out
}

func main() {
	defer func() {
		fmt.Println("the number of goroutines: ", runtime.NumGoroutine())
	}()

	out := gen(2, 3)

	for n := range out {
		fmt.Println(n)              // 2
		time.Sleep(5 * time.Second) // done thing, 可能异常中断接收
		if true {                   // if err != nil
			break
		}
	}
}

```
main函数发生异常，导致处理中断，退出循环。gen 函数中启动的 goroutine 并不会退出。
```sh
gary@gary-asus:~/project/test/gochan$ go run gochan.go 
2
the number of goroutines:  2
```

问题分析：

当接收者停止工作，发送者并不知道，仍阻塞等待向下游发送数据。Golang 可以通过 channel 的关闭向所有的接收者发送广播信息。

问题修改：
```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func gen(done chan struct{}, nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			select {
			case out <- n:
			case <-done:
				return
			}
		}
	}()
	return out
}

func main() {
	defer func() {
		time.Sleep(time.Second)
		fmt.Println("the number of goroutines: ", runtime.NumGoroutine())
	}()

	done := make(chan struct{})
	defer close(done)

	out := gen(done, 2, 3)

	for n := range out {
		fmt.Println(n)              // 2
		time.Sleep(5 * time.Second) // done thing, 可能异常中断接收
		if true {                   // if err != nil
			break
		}
	}
}

```
```sh
gary@gary-asus:~/project/test/gochan$ go run gochan.go 
2
the number of goroutines:  1
```

##### 2. channel 接收不发送
```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	defer func() {
		time.Sleep(time.Second)
		fmt.Println("the number of goroutines: ", runtime.NumGoroutine())
	}()

	var ch chan struct{}
	go func() {
		ch <- struct{}{}
	}()
}
```
```sh
gary@gary-asus:~/project/test/gochan$ go run gochan.go 
the number of goroutines:  2
```

##### 3. channel nil

向 nil channel 发送和接收数据都将会导致阻塞
```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	defer func() {
		time.Sleep(time.Second)
		fmt.Println("the number of goroutines: ", runtime.NumGoroutine())
	}()

	var ch chan int
	go func() {
		<-ch
		// ch<-
	}()
}

```
```sh
gary@gary-asus:~/project/test/gochan$ go run gochan.go 
the number of goroutines:  2
```

问题修改：
```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	defer func() {
		time.Sleep(time.Second)
		fmt.Println("the number of goroutines: ", runtime.NumGoroutine())
	}()

	done := make(chan struct{})

	var ch chan int
	go func() {
		defer close(done)
	}()

	select {
	case <-ch:
	case <-done:
		return
	}
}
```
```sh
gary@gary-asus:~/project/test/gochan$ go run gochan.go 
the number of goroutines:  1
```

##### 4. sync.Mutex
```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func main() {
	total := 0

	defer func() {
		time.Sleep(time.Second)
		fmt.Println("total: ", total)
		fmt.Println("the number of goroutines: ", runtime.NumGoroutine())
	}()

	var mutex sync.Mutex
	for i := 0; i < 2; i++ {
		go func() {
			mutex.Lock()
			total += 1
		}()
	}
}
```
为防止出现数据竞争，对计算部分做了加锁保护，但并没有及时解锁，导致 `i = 1` 的 goroutine 一直阻塞等待 `i = 0` 的 goroutine 释放锁
```sh
gary@gary-asus:~/project/test/gomutex$ go run gomutex.go 
total:  1
the number of goroutines:  2
```

问题修改： `defer mutex.UnLock()`

##### 5. sync.WaitGroup

WaitGroup 和锁有所不同，它类似 Linux 中的信号量，可以实现一组 goroutine 操作的等待。使用的时候，如果设置了错误的任务数，也可能会导致阻塞导致泄露发生。

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func handle() {
	var wg sync.WaitGroup

	wg.Add(4)

	go func() {
		fmt.Println("访问表1")
		wg.Done()
	}()

	go func() {
		fmt.Println("访问表2")
		wg.Done()
	}()

	go func() {
		fmt.Println("访问表3")
		wg.Done()
	}()

	wg.Wait()
}

func main() {
	defer func() {
		time.Sleep(time.Second)
		fmt.Println("the number of goroutines: ", runtime.NumGoroutine())
	}()

	go handle()
	time.Sleep(time.Second)
}

```

```sh
gary@gary-asus:~/project/test/gowait$ go run gowait.go 
访问表1
访问表3
访问表2
the number of goroutines:  2
```

问题修改：尽量不要一次设置全部任务数，即使数量非常明确的情况。因为在开始多个并发任务之间或许也可能出现代码异常被阻断的情况发生。最好是在任务启动时通过 `wg.Add(1)` 的方式增加。

```go
    wg.Add(1)
    go func() {
        fmt.Println("访问表1")
        wg.Done()
    }()
 
    wg.Add(1)
    go func() {
        fmt.Println("访问表2")
        wg.Done()
    }()
 
    wg.Add(1)
    go func() {
        fmt.Println("访问表3")
        wg.Done()
    }()
```



