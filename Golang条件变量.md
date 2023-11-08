---
title: Golang条件变量
created: '2023-11-08T08:31:10.220Z'
modified: '2023-11-08T13:30:47.543Z'
---

# Golang条件变量

`sync.Cond`是Go语言标准库中的一个类型，代表条件变量。条件变量是用于多个goroutine之间进行同步和互斥的一种机制。`sync.Cond`可以用于等待和通知goroutine，以便它们可以在特定条件下等待或继续执行。

`sync.Cond`的定义如下，提供了`Wait` ,`Singal`,`Broadcast`以及`NewCond`方法
```go
type Cond struct {
   noCopy noCopy
   // L is held while observing or changing the condition
   L Locker

   notify  notifyList
   checker copyChecker
}

func NewCond(l Locker) *Cond {}
func (c *Cond) Wait() {}
func (c *Cond) Signal() {}
func (c *Cond) Broadcast() {}
```

- `NewCond`方法： 提供创建Cond实例的方法 
- `Wait`方法: 使当前线程进入阻塞状态，等待其他协程唤醒 
- `Singal`方法: 唤醒一个等待该条件变量的协程，如果没有线程在等待，则该方法会立即返回。 
- `Broadcast`方法: 唤醒所有等待该条件变量的协程，如果没有线程在等待，则该方法会立即返回。 


当使用`sync.Cond`时，通常需要以下几个步骤：
- 定义一个互斥锁，用于保护共享数据；
- 创建一个`sync.Cond`对象，关联这个互斥锁；
- 在需要等待条件变量的地方，获取这个互斥锁，并使用`Wait`方法等待条件变量被通知；
- 在需要通知等待的协程时，使用`Signal`或`Broadcast`方法通知等待的协程；
- 最后，释放这个互斥锁。 

简单的代码示例
```go
var (
    // 1. 定义一个互斥锁
    mu    sync.Mutex
    cond  *sync.Cond
    count int
)
func init() {
    // 2.将互斥锁和sync.Cond进行关联
    cond = sync.NewCond(&mu)
}
go func(){
    // 3. 在需要等待的地方,获取互斥锁，调用Wait方法等待被通知
    mu.Lock()
    // 这里会不断循环判断 是否满足条件
    for !condition() {
       cond.Wait() // 等待任务
    }
    mu.Unlock()
}

go func(){
     // 执行业务逻辑
     // 4. 满足条件，此时调用Broadcast唤醒处于等待状态的协程
     cond.Broadcast() 
}
```

条件变量的作用并不保证在同一时刻仅有一个协程访问某个共享数据资源，而是在对应的共享数据的状态发生变化时，通知阻塞在某个条件下的协程。条件变量不是锁，在并发中不能达到同步的目的，因此条件变量总是与锁一块使用。

例如：如果channel缓冲队列满了，可以使用条件变量让生产者对应的goroutine暂停（阻塞）生产数据，但是当消费者读取缓冲队列的数据了，队列状态改变为未满，应该唤醒（发送通知）阻塞的生产者goroutine继续生产数据。

生产者消费者模型示例代码

```go
package main

import(
  "fmt"
  "sync"
  "math/rand"
  "time"
)

var cond sync.Cond // 创建全局条件变量

// 生产者
func producer(out chan<- int, idx int){
  for{
    cond.L.Lock() // 条件变量用互斥锁
    for (len(out) == 3) { // 产品数据满 等待消费者消费
      cond.Wait() // 挂起当前协程，等待条件变量满足后被消费者唤醒
    }
    num := rand.Intn(1000)
    out <-num
    fmt.Printf("%d 生产者 产生数据 %d, 公共区剩余 %d 个数据 \n", idx, num, len(out))
    cond.L.Unlock() // 生产结束，解锁互斥锁
    cond.Signal() // 唤醒阻塞的消费者
    time.Sleep(time.Second)
  }
}

// 消费者
func consumer(in <-chan int, idx int){
  for{
    cond.L.Lock() // 与生产者是同一个
    for len(in) == 0 { // 产品缓冲区为空，等待生产者
      cond.Wait() // 挂起当前协程，等待条件变量满足条件后被生产者唤醒
    }
    num := <-in // 将channel数据消费
    fmt.Printf("%d 消费者 消费数据 %d, 公共区剩余 %d 个数据 \n", idx, num, len(in))
    cond.L.Unlock() // 消费结束，解锁互斥锁
    cond.Signal() // 唤醒阻塞的生产者
    time.Sleep(time.Millisecond * 300) 

  }
}

func main(){
  rand.Seed(time.Now().UnixNano()) // 设置随机数种子
  quit := make(chan bool) // 用于结束通信的channel

  product := make(chan int, 3) // 产品数据(公共数据)使用channel模拟
  cond.L = new(sync.Mutex) // 条件变量的互斥锁

  for i:=0; i < 5; i++ {
    go producer(product, i+1)
  }

  for i:=0; i < 3; i++ {
    go consumer(product, i+1)
  }
  <-quit // 主协程阻塞 不结束
}
```

