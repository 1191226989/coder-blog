---
title: Golang实现进程退出以后自动重启
created: '2023-08-23T06:52:43.394Z'
modified: '2023-08-23T07:11:05.085Z'
---

# Golang实现进程退出以后自动重启

```go
package main

import (
  "log"
  "os"
  "os/exec"
  "syscall"
  "time"
)

func main(){
  for {
    cmd := exec.Command("./your-program")
    // 将子进程的标准输入输出以及错误 连接到父进程的标准输入输出及错误
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    // 启动子进程
    err := cmd.Start()
    if err != nil {
      log.Fatal(err)
    }

    // 监视子进程状态
    err = cmd.Wait()
    if err != nil {
      log.Printf("子进程退出: %s", err)
    }else {
      log.Println("子进程正常退出")
    }

    // 等待一段时间后重启进程
    time.Sleep(5 * time.Second)
  }
}
```

- 使用一个无限循环来监视进程状态并重新启动它。在每次循环中，我们使用exec.Command创建一个新的命令。

- 将子进程的标准输入输出及错误连接到父进程的标准输入输出及错误，以便在启动子进程后仍然可以通过父进程终端进行交互。

- cmd.Start()启动子进程

- cmd.Wait()等待子进程退出，并检查退出状态。如果子进程异常退出，则记录相关日志。

- time.Sleep()在重启进程之前等待一段时间，可以防止进程在短时间内频繁的重启。

