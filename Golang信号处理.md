---
title: Golang信号处理
created: '2023-09-04T08:54:14.250Z'
modified: '2023-09-04T09:10:20.865Z'
---

# Golang信号处理

signal.Notify必须使用 buffered channel 来做监听

```go
package main

import (
	"fmt"
	"os"
	"os/signal"
)

func main() {
  c := make(chan os.Signal, 1)
  signal.Notify(c, os.Interrupt)

  // Block until a signal is received
  s := <-c
  fmt.Println("Got signal:", s)
}
```

如果将之前的 buffered channel 修改为 unbuffered，同样会得到与使用 buffered channel 同样的结果

```go
package main

import (
	"fmt"
	"os"
	"os/signal"
)

func main() {
  c := make(chan os.Signal)
  signal.Notify(c, os.Interrupt)

  // Block until a signal is received
  s := <-c
  fmt.Println("Got signal:", s)
}
```

对程序做一些修改，让 main() 程序在接收中断前做一些其他比较耗时的任务，例如 time.sleep，就会引发问题:

发现在前五秒内，无论执行多少次 Ctrl+c，程序都不会中止，等到五秒结束后，需要再次按 Ctrl+c 程序才会中止。这与预期的在五秒内执行一次 Ctrl+c 之后到五秒时程序就会自动中止的结果差异很大，说明在五秒内的系统中断都没有被监听到。

```go
package main

import(
  "fmt"
  "os"
  "os/signal"
)

func main(){
  c := make(chan os.Signal)
  signal.Notify(c, os.Interrupt)

  time.Sleep(time.Second * 5)

  // Block until a signal is received
  s := <-c
  fmt.Println("Got signal:", s)
}
```

问题在原因在于 `signal.go` 的 process 函数，其中处理消息部分有如下内容：
```go
for c, h := range handlers.m{
  if h.want(n){
    // send but do not block for it
    select {
      case c <- sig:
      // ...
      default:
    }
  }
}
// 由于 unbuffered channel 长度为0，造成在五秒内收到的任何消息都会进入 default 条件，造成 channel 中没有任何值，从而接收不到中断消息信号
```

