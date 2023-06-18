---
title: Golang并发编程
created: '2023-06-01T14:42:18.587Z'
modified: '2023-06-05T14:48:28.328Z'
---

# Golang并发编程

#### 进程和线程
- 进程是程序在操作系统中的一次执行过程，系统进行资源分配和调度的一个独立单位。
- 线程是进程的一个执行实体，是 CPU 调度和分派的基本单位，它是比进程更小的能独立运行的基本单位。
- 一个进程可以创建和撤销多个线程，同一个进程中的多个线程之间可以并发执行。

#### 并发和并行
- 多线程程序在一个核心的 CPU 上运行，就是并发
```
 ^
A|A..A..A..A..........>
B|.B..B..B..B.........>
C|..C..C..C..C........>
 |————————————————————>
 Concurrency: 1 single processor
```

- 多线程程序在多个核心的 CPU 上运行，就是并行
```
 ^
A|AAAAAAAAAAAAAAAAA...>
B|BBBBBBBBBBBBBBBBB...>
C|CCCCCCCCCCCCCCCCC...>
 |————————————————————>
 Parallelism: 1 multiprocessores, multicore
```
> 并发不是并行：
> + 并发主要由切换时间片来实现 “同时” 运行，并行则是直接利用多核实现多线程的运行
> + go语言可以设置使用核数，以发挥多核计算机的能力

#### 协程和线程
- 协程：独立的栈空间，共享堆空间，调度由用户自己控制。
> 本质上有点类似于用户级线程，这些用户级线程的调度也是自己实现的。

- 线程：一个线程上可以跑多个协程，协程是轻量级的线程。
> goroutine 只是官方实现的超级线程池。每个实例 4 ~ 5 KB 的栈内存占用，以及由于实现机制从而大幅减了的创建和销毁开销。

#### goroutine

C++ 实现并发编程需要开发者手动维护一个线程池，需要手动调度线程并维护上下文切换。Go 语言在语言层面已经内置了调度和上下文切换机制，不需要开发者去维护。

在 Go 中需要让某个任务并发执行，只需要将其包装成一个函数，开启一个 goroutine 去执行该函数。

如果主协程退出，其他任务就不会继续执行了。

OS 线程（操作系统线程）一般都有固定的栈内存（通常为 2MB）

一个 goroutine 的栈在其生命周期开始时只有很小（典型情况下 2KB）。goroutine 的栈内存不固定，可以按需增大缩小，最大限制可以达到 1GB。所以 Go 语言中可以一次创建 10万左右的 goroutine

#### runtime 包
Go 语言中操作系统线程和 goroutine 的关系：
- 一个操作系统线程对应多个 goroutine
- go 程序可以同时使用多个操作系统线程
- goroutine 和 OS 线程是多对多关系（ m : n ）

- runtime.Gosched()让出 CPU 时间片，重新等待安排任务，允许其他 goroutine 运行，但不会结束当前 goroutine
```go
func main(){
  go func(s string){
    for i:=0; i<2; i++{
      fmt.Println(s)
    }
  }("world")
  // 主协程
  for i:=0;i<2;i++{
    // 让出时间片，重新等待安排任务
    runtime.Gosched()
    fmt.Println("hello")
  }
}

```
- runtime.Goexit()退出当前协程。
```go
func main(){
  go func(){
    defer fmt.Println("A.defer")
    func(){
      defer fmt.Println("B.defer")
      // 结束协程
      runtime.Goexit()
      defer fmt.Println("C.defer")
      fmt.Println("B")
    }()
    fmt.Println("A")
  }()
  time.Sleep(time.Second)
}
```
- runtime.GOMAXPROCS() Go 运行时的调度器使用 `GOMAXPROCS` 参数来确定需要使用多少个 OS 线程来同时执行 Go 代码。

默认值是机器上的 CPU 核心数。例如在一个 8 核机器上，调度器会把 Go 代码同时调度到 8 个 OS 线程上。
> GOMAXPROCS 是 m:n 调度中的 n

通过 `runtime.GOMAXPROCS()` 函数设置当前程序并发时占用的 CPU 逻辑核心数。
> Go 1.5 之前，默认使用单核执行。Go 1.5 以后，默认使用全部的 CPU 逻辑核心数。

逻辑核心设置为 1，则只有单核在执行（可以并发不能并行）。
```go
func a(){
  for i:=1;i<10000;i++{
    fmt.Println("A:", i)
  }
}

func b(){
  for i:=1;i<10000;i++{
    fmt.Println("B:", i)
  }
}

func main(){
  runtime.GOMAXPROCS(1)
  go a()
  go b()
  time.Sleep(time.Second)
}
```
设置 2 个逻辑核心，则两个任务并行执行
```go
func a(){
  for i:=1;i<10000;i++{
    fmt.Println("A:", i)
  }
}

func b(){
  for i:=1;i<10000;i++{
    fmt.Println("B:", i)
  }
}

func main(){
  runtime.GOMAXPROCS(2)
  go a()
  go b()
  time.Sleep(time.Second)
}
```

#### channel
> 单纯地将函数并发执行是没有意义的。函数与函数间需要交换数据才能体现并发执行的意义

虽然可以使用共享内存进行数据交换，但是共享内存在不同的 goroutine 中容易发生竞态问题。为了保证数据交换的正确性，必须使用互斥量对内存进行加锁，这种做法势必造成性能问题。

Go 语言的并发模型是 CSP，提倡**通过通信共享内存而不是通过共享内存而实现通信**
> CSP：Communicating Sequential Processes，通信顺序进程

channel有发送（send）、接收(receive）和关闭（close）三种操作。

关闭通道注意事项：
- 只有在通道接收方 goroutine 所有的数据都接收完毕时才需要关闭通道，通道是可以被垃圾回收的。
- 和关闭文件不一样，结束操作后关闭文件是必须要做的，但是关闭通道不是必须的

关闭后的通道有以下特点：
- 对一个关闭的通道再发送值就会导致 panic
- 对一个关闭的通道进行接收会一直获取值直到通道为空
- 对一个关闭的并且没有值的通道执行接收操作会得到对应类型的零值
- 关闭一个已经关闭的通道会导致 panic

- 无缓冲的channel
无缓冲的通道又被称为阻塞的通道
```go
func main(){
  ch := make(chan int)
  ch<-10
  fmt.Println("发送成功")
}
// 上面代码可以通过编译，但执行时会报错：deadlock
// fatal error: all goroutines are asleep - deadlock!
```
由于使用 `ch := make(chan int)` 创建的是无缓冲的通道，无缓冲的通道必须有接收才能发送

```go
// 启用一个goroutine接收数据
func receive(c chan int){
  ret := <-c
  fmt.Println("接收成功", ret)
}

func main(){
  ch := make(chan int)
  go receive(ch)
  ch<-10
  fmt.Println("发送成功")
}
// 发送操作先执行则会阻塞，直到另一个 goroutine 在该通道上执行接收操作，才能发送成功
// 接收操作先执行，则接收方的 goroutine 将阻塞，直到另一个 goroutine 在该通道上发送一个值
// 使用无缓冲通道进行通信将导致发送和接收的 goroutine 同步化，因此无缓冲通道也称为同步通道
```

#### 有缓冲的channel
可以在使用 make 函数初始化通道的时候为其指定通道的容量

通过 close() 函数关闭 channel，如果不再往通道里发送值或者取值的时候，记得关闭通道。

如何判断一个通道是否被关闭：
1. 方法一：通过从通道取值操作的第 2 个返回值 ok 来判断，通道关闭再取值则 ok = false
2. 方法二：通过 for range 循环，通道关闭后会自动退出循环

```go
func main(){
  ch1 := make(chan int)
  ch2 := make(chan int)

  // 开启goroutine将0-100发送到ch1
  go func(){
    for i:=0; i<100; i++{
      ch1 <-i
    }
    close(ch1)
  }()

  // 开启goroutine从ch1中接收值，并将该值的平方值发送到ch2
  go func(){
    for {
      i, ok := <-ch1
      if !ok {
        break
      }
      ch2 <- i*i
    }
    close(ch2)
  }()

  // 主协程从ch2接收数据，channel关闭后会退出for range循环
  for i:= range ch2 {
    fmt.Println(i)
  }
}
```

#### 单向channel
```go
// chan<- int 只能发送数据不能接收
func couter(out chan<- int){
  for i:=0; i<100; i++{
    out <- i
  }
  close(out)
}

// <-chan int 只能接收数据不能发送
func squarer(out chan <- int, in <-chan int){
  for i := range in{
    out <- i*i
  }
  close(out)
}

func printer(in <-chan int){
  for i := range in{
    fmt.Println(i)
  }
}

func main(){
  ch1 := make(chan int)
  ch2 := make(chan int)

  go couter(ch1)
  go squarer(ch2, ch1)

  printer(ch2)
}
```

#### goroutine池
worker pool
- 本质上是生产者消费者模型
- 可以有效控制goroutine数量

```go
type Job struct{
  Id int
  RandNum int
}

type Result struct{
  job *Job
  sum int
}

func main(){
  // 1. job channel
  jobChan := make(chan *Job, 128)
  // 2. 结果 channel
  resultChan := make(chan *Result, 128)
  // 3. 创建工作池
  createPool(64, jobChan, resultChan)
  // 4. 打印结果
  go func(resultChan chan *Result){
    for result := range resultChan{
      fmt.Printf("job id:%v randnum:%v result:%d \n", result.job.Id, result.job.RandNum, result.sum)
    }
  }(resultChan)

  var id int
  // 循环创建job
  for {
    id++
    randNum := rand.Int()
    job := &Job{
      Id: id,
      RandNum: randNum,
    }
    jobChan<- job
  }
}

// 创建工作池
func createPool(num int, jobChan chan *Job, resultChan chan *Result){
  for i:=0; i<num; i++{
    go func(jobChan chan *Job, resultChan chan *Result){
      for job := range jobChan{
        randNum := job.RandNum
        sum := sumAll(randNum)
        ret := &Result{
          job: job,
          sum: sum,
        }
        // 运算结果发送到channel
        resultChan<- ret
      }
    }
  }
}

func sumAll(num int) int{
  var sum int
  for num != 0 {
    tmp := num % 10
    sum += tmp
    num = num / 10
  }
  return sum
}
```

#### 读写互斥锁

读写锁分为两种：读锁、写锁。
- 当一个 goroutine 获取 读锁 后，其他 goroutine 如果获取到 读锁 则继续获得，获取 写锁 则等待
- 当一个 goroutine 获取 写锁 后，其他 goroutine 无论获取 读锁 还是 写锁 都会等待
```go
var (
  x int64
  wg sync.WaitGroup
  rwlock sync.RWMutex
)

func write(){
  rwlock.Lock()// 写锁
  x = x + 1

  time.Sleep(10 * time.Millisecond)
  rwlock.Unlock
  wg.Done()
}

func read(){
  rwlock.RLock() // 读锁
  time.Sleep(time.Millisecond)
  rwlock.RUnlock()
  wg.Done()
}

func main(){
  start := time.Now()
  for i:=0;i<10;i++{
    wg.Add(1)
    go write()
  }
  for i:=0;i<1000;i++{
    wg.Add(1)
    go read()
  }

  wg.Wait()
  end := time.Now()
  fmt.Println(end.Sub(start))
}
```




















