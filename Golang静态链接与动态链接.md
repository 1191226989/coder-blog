---
title: Golang静态链接与动态链接
created: '2023-09-05T06:19:54.038Z'
modified: '2023-09-05T07:27:06.809Z'
---

# Golang静态链接与动态链接

```sh
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags '-s -w --extldflags "-static -fpic"' main.go

```
- CGO_ENABLED 这个参数默认为1，开启CGO
- GOOS 和 GOARCH 用来指定要构建的平台为Linux
- 可选参数 `-ldflags` 是编译选项：
  - `-s -w` 去掉调试信息，可以减小构建后文件体积
  - `--extldflags "-static -fpic"` 完全静态编译，编译生成的文件可以任意放到指定平台下运行，而不需要运行环境配置

1. Go语言在默认情况下是静态链接的
```sh
gary@gary-asus:~/project/test/linkfile$ cat linkfile.go 
package main

import "fmt"

func main() {
        fmt.Println("Hello World")
}

gary@gary-asus:~/project/test/linkfile$ go build
gary@gary-asus:~/project/test/linkfile$ ls
go.mod  linkfile  linkfile.go
gary@gary-asus:~/project/test/linkfile$ ./linkfile 
Hello World
gary@gary-asus:~/project/test/linkfile$ file linkfile
linkfile: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=VvaLfyr3FE7ZBE8vVO2s/F2iIVakIVDKei8UP4rwT/7kNG0Fjgw8h_HUyJUuAa/RRSmNytvDctVa5W6roBk, with debug_info, not stripped
gary@gary-asus:~/project/test/linkfile$ ldd linkfile
        不是动态可执行文件

```

2. 有一些库可能会导致动态链接，例如："net/http"
```sh
gary@gary-asus:~/project/test/linkfile$ cat httpget.go 
package main

import (
        "fmt"
        "net/http"
)

func main() {
        r, err := http.Get("http://www.example.com")
        if err != nil {
                fmt.Println(err)
        }
        fmt.Println(r.StatusCode)
}
gary@gary-asus:~/project/test/linkfile$ go build -o httpget httpget.go 
gary@gary-asus:~/project/test/linkfile$ ./httpget 
200
gary@gary-asus:~/project/test/linkfile$ file httpget
httpget: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=1kSW7LFPK_SH_QAEAyh1/_MyllxN_5_RHuj9aCWgu/N6HELloUSZ9eTYg7PEMX/08XjF-SI1dcoJfeTBx4R, with debug_info, not stripped
gary@gary-asus:~/project/test/linkfile$ ldd httpget
        linux-vdso.so.1 (0x00007ffd3f1f4000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f1e2d600000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f1e2d9e7000)
```
编译时可以增加 `-ldflags="-extldflags -static"` 参数进行静态链接
```sh
go build -ldflags="-extldflags -static"
```
有些版本 Ubuntu 上使用 Go 进行静态链接需要禁用CGO
```sh
CGO_ENABLED=0 go build -o app main.go

这会禁用 CGO（C语言的调用接口），从而强制使用静态链接。
```

```sh
gary@gary-asus:~/project/test/linkfile$ CGO_ENABLED=0 go build -o httpget httpget.go 
gary@gary-asus:~/project/test/linkfile$ ldd httpget
        不是动态可执行文件
gary@gary-asus:~/project/test/linkfile$ file httpget 
httpget2: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=kMgrsFvOVEhE2JTw-LPi/vl3k02CwXhJ0ZglORlAE/y7cM1o3QZEml_Rt8mKzB/4GMekWLr1reMvsynDQGh, with debug_info, not stripped
```

3. 存在一些第三方库，因调用了一些 `glibc` 中不支持静态链接的函数，导致无法静态链接
```sh
gary@gary-asus:~/project/test/linkfile$ cat gorm.go 
package main

import (
        "gorm.io/driver/sqlite"
        "gorm.io/gorm"
)

type Product struct {
        gorm.Model
        Code  string
        Price uint
}

func main() {
        db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
        if err != nil {
                panic("failed to connect database")
        }

        db.AutoMigrate(&Product{})
}

gary@gary-asus:~/project/test/linkfile$ go build -o gorm -ldflags '-s -w --extldflags "-static -fpic"' gorm.go 
# command-line-arguments
/usr/bin/ld: /tmp/go-link-3894712155/000011.o: in function `unixDlOpen':
/home/gary/go/pkg/mod/github.com/mattn/go-sqlite3@v1.14.17/sqlite3-binding.c:43784: 警告： Using 'dlopen' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
```
这类情况，如果需要静态链接，那么可以弃用 `glibc` 库，改用 `musl libc`库[https://www.musl-libc.org/](https://www.musl-libc.org/) 

操作系统 Debian / Ubuntu ，安装 musl libc 库：
```sh
sudo apt-get install musl-dev musl-tools
```

使用musl libc库静态链接编译成功
```sh
gary@gary-asus:~/project/test/linkfile$ CC=musl-gcc go build -o gorm -tags musl -ldflags '-s -w --extldflags "-static -fpic
"' gorm.go 
gary@gary-asus:~/project/test/linkfile$ file gorm
gorm: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=Wzea57YkdWwkOzR2lvWI/CipsnhxMBYf4jbk2WYo4/2cvjs3CdO34GYj8MZLXw/D2K7P4Xg4D72iwYHxBCj, stripped
gary@gary-asus:~/project/test/linkfile$ ldd gorm
        不是动态可执行文件
```


