---
title: Golang控制协程并发数量
created: '2023-11-10T14:09:14.105Z'
modified: '2023-11-13T09:53:14.443Z'
---

# Golang控制协程并发数量

#### 方法1：使用有缓冲容量长度的channel控制
```go
package main

import (
	"fmt"
	"time"
)

//同时最多10个协程运行
var limitMaxNum = 10
var chData = make(chan int, limitMaxNum)

//有100个任务要处理
var tasknum = 100

//使用 有缓冲容量长度的channel
func main() {
	var i, j int
	var chanRet = make(chan int, tasknum) //运行结果存储到chanRet

	//运行处理
	go func() {
		for i = 0; i < tasknum; i++ {
			chData <- 1
			go dotask(i, chanRet)
		}
	}()

	//获取返回结果
	for j = 0; j < tasknum; j++ {
		<-chData
		ret ：= <-chanRet
		fmt.Println("ret:", ret)
	}
	fmt.Println("main over")
}

func dotask(taskid int, chanRet chan int) {
	time.Sleep(time.Millisecond * 100)
	fmt.Println("finish task ", taskid)

	chanRet <- taskid * taskid
}
```

#### 方法2：使用channel+waitGroup
```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var limitMaxNum = 10
var chData = make(chan int, limitMaxNum)
var jobGroup sync.WaitGroup
var tasknum = 100

//使用有缓冲容量长度的channel
func main() {
	var i int
	//var chanRet = make(chan int, tasknum)

	//处理任务，最多同时有10个协程
	for i = 0; i < tasknum; i++ {
		chData <- 1
		go dotask(i)
	}

	//使用Wait等待所有任务执行完毕
	jobGroup.Wait()
	fmt.Println("main over")
}

func dotask(taskid int) {
	jobGroup.Add(1)

	time.Sleep(time.Millisecond * 100)
	fmt.Println("finish task ", taskid)

	<-chData

	jobGroup.Done()
}
```

