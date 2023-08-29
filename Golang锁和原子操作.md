---
title: Golang锁和原子操作
created: '2023-08-29T08:26:48.779Z'
modified: '2023-08-29T09:20:36.933Z'
---

# Golang锁和原子操作

在编码中会存在多个 goroutine 协程同时操作一个资源（临界区）发生数据竞争，也就是 `race` 竞态问题。

```go
package main
import (
  "fmt"
  "sync"
)

var num int64
var wg sync.WaitGroup

func add() {
  for i :=0; i < 1000000; i++ {
    num = num + 1
  }
  wg.Done()
}

func main() {
  wg.add(2)

  go add()
  go add()

  wg.Wait()
  fmt.Println(num)
}
```
多次运行代码，得到的结果都不一样。原因分析：两个 goroutine 协程在访问和修改 num 变量，会存在同时对同一个值的 num +1 的情况，最终 num 总共只加了 1 而不是 2。

#### 锁

1. **互斥锁**

加锁后上面代码的运行过程：
```go
package main
import (
  "fmt"
  "sync"
)

var num int64
var wg sync.WaitGroup
var lock sync.Mutex

func add() {
  for i :=0; i < 1000000; i++ {
    lock.Lock()
    num = num + 1
    lock.Unlock()
  }
  wg.Done()
}

func main() {
  wg.add(2)
  
  go add()
  go add()

  wg.Wait()
  fmt.Println(num)
}
```
```
协程1开始第一次循环
协程2开始第一次循环
协程1给num加互斥锁
协程2给num加互斥锁（因为协程1已经锁定了num，所以协程2开始阻塞）
协程1执行num=num+1 num = 1
协程1释放互斥锁
协程2被唤醒 并加锁成功
协程1进入第二次循环
协程2执行num=num+1 num = 2
协程1给num加互斥锁（因为协程2已经锁定了num，所以协程1开始阻塞）
协程2释放互斥锁
协程1被唤醒并加锁成功
...
```
num 只有一个，只要被锁定，其他访问者不管是读还是写都无法访问。只有等待解锁后去抢占锁才能访问。

互斥锁一般被使用在 **写大于读** 的场景。互斥锁是一种常用的控制共享资源访问的方法，它能保证同时只有一个 goroutine 协程可以访问共享资源。

2. **读写锁**

```go
package main

import (
   "fmt"
   "sync"
   "time"
)

var (
   num    int64
   wg     sync.WaitGroup
   rwlock sync.RWMutex
)

func write() {
   rwlock.Lock()

   num = num + 1
   // 模拟真实写数据消耗的时间
   time.Sleep(10 * time.Millisecond)

   rwlock.Unlock()
   wg.Done()
}

func read() {
   rwlock.RLock()

   // 模拟真实读取数据消耗的时间
   time.Sleep(time.Millisecond)

   rwlock.RUnlock()
   wg.Done()
}

func main() {
   // 用于计算时间 消耗
   start := time.Now()

   // 开5个协程用作 写
   for i := 0; i < 5; i++ {
      wg.Add(1)
      go write()
   }

   // 开500 个协程，用作读
   for i := 0; i < 1000; i++ {
      wg.Add(1)
      go read()
   }

   // 等待子协程退出
   wg.Wait()
   end := time.Now()

   // 打印程序消耗的时间
   fmt.Println(end.Sub(start))
}
```
读写混合的场景下,假如 G1、G3 只读， G2、G4 只写

```txt
G1加读锁 成功
G2加写锁（资源已被读锁定，自旋）
G3加读锁 成功
G4加写锁（资源已被读锁定，自旋）
G1解读锁
G3解读锁
G2加写锁成功（G2持续尝试加写锁，现在没有读锁，所以加锁成功）
G1加读锁（资源已被写锁定，自旋）
G2解写锁
G4加写锁成功（G4持续尝试加写锁，现在没有读锁，所以加锁成功）
G4解写锁
G1加读锁 成功
```
当一个 goroutine 协程获取 **读锁** 之后，其他的 goroutine 协程如果是获取读锁会继续获得锁；如果是获取写锁就必须等待。当一个 goroutine 协程获取 **写锁** 之后，其他的 goroutine 协程无论是获取读锁还是写锁都会等待。

> 读写锁实际是一种特殊的自旋锁,它把对共享资源的访问者划分成读者和写者,同一时间可以多线程对同一共享资源的访问,但是不能同时对该资源数据进行修改。

写者是排他性的，一个读写锁同时只能有一个写者或多个读者（与CPU数相关），但不能同时既有读者又有写者。在读写锁保持期间也是抢占失效的。

如果读写锁当前没有读者，也没有写者，那么写者可以立刻获得读写锁，否则它必须自旋在那里，直到没有任何写者或读者。

如果读写锁没有写者，那么读者可以立即获得该读写锁，否则读者必须自旋在那里，直到写者释放该读写锁。

- 写锁是排他性的，一个读写锁同时只能有一个写或者多个读
- 不能同时既有读又有写
- 如果资源未被读写锁锁定，那么写者可以即刻获得写锁。否则它必须原地自旋，直到资源被所有者释放
- 如果资源未被写者锁定，那么读者可以立刻获得读锁，否则读者必须原地自旋，直到写者释放写锁

3. **自旋锁**

自旋锁的概念很简单，就是如果加锁失败在等锁时是使用休眠等待还是忙等待，如果是忙等待的话，就是自旋锁。

> 自旋也叫自旋锁，是专门为了防止多处理器并发而引入的一种锁，它在内核中大量应用于中断处理等部分（对于单处理器来说，防止中断处理中的并发可简单采用关闭中断的方式，即在标志寄存器中关闭 / 打开中断标志位，不需要自旋锁）。

自旋就是在并发过程中，若其中一个协程拿不到锁，会不停的去尝试拿锁，而不是阻塞睡眠。

- 互斥锁：当拿不到锁的时候，会阻塞等待，会睡眠，等待锁释放后被唤醒。（写多读少）

- 读写锁：当拿不到锁的时候，会在原地不停的看能不能拿到锁，所以叫自旋锁，不会阻塞，不会睡眠。（读多写少）

- 如果读和写都很多，就需要用原子操作

自旋锁是用于多线程同步的一种锁，线程反复检查锁变量是否可用。由于线程在这一过程中保持执行，因此是一种忙等待。一旦获取了自旋锁，线程会一直保持该锁，直至显式释放自旋锁。自旋锁避免了进程上下文的调度开销，因此对于线程只会阻塞很短时间的场合是有效的。

#### 原子操作

> "原子操作 (atomic operation) 不需要 synchronized"。所谓原子操作是指不会被线程调度机制打断的操作。这种操作一旦开始就一直运行到结束，中间不会有任何 context switch （切换到另一个线程）。

原子操作是不可分割的，在执行完毕之前不会被任何其他任务或事件中断

加锁操作会涉及到内核态的上下文切换会比较耗时，代价比较高。针对基本的数据类型我们可以使用原子操作来保证并发安全

因为原子操作是 Go 语言提供的方法，在用户态就可以完成，执行效率比加锁操作更好，不用自己写汇编，Go 官方库通过 sync/atomic 包提供了原子操作支持

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
    "time"
)

var num int64
var l sync.Mutex
var wg sync.WaitGroup

// 普通版加函数
func add() {
    num = num + 1
    wg.Done()
}

// 互斥锁版加函数
func mutexAdd() {
    l.Lock()
    num = num + 1
    l.Unlock()
    wg.Done()
}

// 原子操作版加函数
func atomicAdd() {
    atomic.AddInt64(&num, 1)
    wg.Done()
}

func main() {
    // 目的是 记录程序消耗时间
    start := time.Now()
    for i := 0; i < 2000000; i++ {

        wg.Add(1)

        // go add()       // 无锁的  add函数 不是并发安全的
        // go mutexAdd()  // 互斥锁的 add函数 是并发安全的，因为拿不到互斥锁会阻塞，所以加锁性能开销大

        go atomicAdd()    // 原子操作的 add函数 是并发安全，性能优于加锁的
    }

    // 等待子协程 退出
    wg.Wait()

    end := time.Now()
    fmt.Println(num)
    // 打印程序消耗时间
    fmt.Println(end.Sub(start))
}
```

无锁情况下耗时 745.292115ms，不是并发安全的，num 结果为 1999848

互斥锁情况下耗时 846.407091ms 并发安全，比无锁要慢

原子操作情况下耗时 806.684619ms 并发安全，比无锁慢，但是比加锁要快

**原子操作函数的第一个参数都是指针**，是因为原子操作需要知道该变量的内存地址，然后以**特殊的 CPU 指令**操作，对于不能取得内存地址的变量是无法进行原子操作的。






