---
title: Golang关闭协程3种方式
created: '2023-07-20T12:40:59.913Z'
modified: '2023-07-20T12:47:01.158Z'
---

# Golang关闭协程3种方式

#### 一、使用 channel 实现协程关闭
需要定义一个 bool 类型的 channel 来控制协程的关闭，并在协程中不断地检测这个 channel 的状态；当 channel 被关闭时，协程就会退出。
```golang
package main

import (
    "fmt"
    "time"
)

func worker(stop chan bool) {
    for {
        select {
        case <-stop:
            fmt.Println("worker stopped")
            return
        default:
            fmt.Println("working...")
            time.Sleep(1 * time.Second)
        }
    }
}

func main() {
    stop := make(chan bool)
    go worker(stop)
    time.Sleep(5 * time.Second)
    fmt.Println("stop worker")
    close(stop)
    time.Sleep(5 * time.Second)
    fmt.Println("program exited")
}
```
#### 二、使用 context 实现协程取消
Context 提供了一种标准的方法，允许传递运行协程的超时、取消信号和请求范围上的其他值。
```golang
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("worker canceled")
            return
        default:
            fmt.Println("working...")
            time.Sleep(1 * time.Second)
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    go worker(ctx)
    time.Sleep(5 * time.Second)
    fmt.Println("cancel worker")
    cancel()
    time.Sleep(5 * time.Second)
    fmt.Println("program exited")
}

```
#### 三、使用 sync.WaitGroup 实现协程等待
在协程启动时，会将 WaitGroup 的计数器加 1；而在协程退出时，会将计数器减 1。当计数器为 0 时，表明所有协程都已经退出，主函数可以继续执行。
```golang
package main

import (
    "fmt"
    "sync"
    "time"
)

func worker(wg *sync.WaitGroup, stop chan bool) {
    defer wg.Done()
    for {
        select {
        case <-stop:
            fmt.Println("worker stopped")
            return
        default:
            fmt.Println("working...")
            time.Sleep(1 * time.Second)
        }
    }
}

func main() {
    wg := sync.WaitGroup{}
    stop := make(chan bool)
    wg.Add(1)
    go worker(&wg, stop)
    time.Sleep(5 * time.Second)
    fmt.Println("stop worker")
    stop <- true
    wg.Wait()
    fmt.Println("program exited")
}

```


