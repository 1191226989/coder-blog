---
title: Golang命令行Cmd参数错误问题
created: '2023-09-05T08:22:33.321Z'
modified: '2023-09-05T08:24:31.841Z'
---

# Golang命令行Cmd参数错误问题

```go
package main

import (
	"fmt"
	"log"
	"os/exec"
)

func main() {

	cmd := &exec.Cmd{
		Path:   "/usr/local/bin/aws",
		Args:   []string{"", "s3", "help"},
		Stdout: log.Writer(),
		Stderr: log.Writer(),
	}
  // Args holds command line arguments, including the command as Args[0].
	// If the Args field is empty or nil, Run uses {Path}.
	//
	// In typical use, both Path and Args are set by calling Command.
	// Args []string
  
	// 如果已声明Cmd.Path = "/usr/local/bin/aws"
	// 则必须像这样声明Cmd.Args = []string{"", "s3", "help"}，因为Args包含上述文档链接中的命令Args[0]
	// 如果声明Cmd.Args = []string{"s3", "help"}，则会提示命令执行参数错误

	fmt.Printf("Command: %s\n", cmd.String())

	err := cmd.Run()
	if err != nil {
		fmt.Println(err)
	}
}
```
