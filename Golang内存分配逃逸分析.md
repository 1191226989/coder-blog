---
title: Golang内存分配逃逸分析
created: '2023-12-01T07:22:46.007Z'
modified: '2023-12-06T02:36:51.023Z'
---

# Golang内存分配逃逸分析

#### 一.什么是内存逃逸
程序在运行过程中，每一个函数都会有自己的内存区域存储自己的局部变量和返回值，这些内存会由编译器在栈中进行分配，每一个函数会分配一个栈帧，在函数运行结束后销毁。但是有些变量在函数运行结束后仍然使用，就需要把这个变量分配在堆上，这种从“栈”上逃逸到“堆”上的现象叫做**内存逃逸**。

- 堆内存(Heap)：一般是手动申请、分配、释放。一般硬件内存有多大堆内存就有多大。适合不可预知大小的内存分配，分配速度较慢，而且会形成内存碎片。

- 栈内存(Stack):是一种拥有特殊规则的线性表数据结构。由编译器进行管理，自动申请、分配、释放。大小一般是固定的。

程序在运行过程中，必不可少的会使用变量、函数和数据，变量和数据在内存中存储的位置可以分为：堆区(Heap)和栈区(Stack)，一般由C或C++编译的程序占用内存分为：栈区、堆区、全局区、常量区、程序代码区。

- 栈区
每个函数都有自己独立的栈空间，函数的**调用参数**、**返回值**以及**局部变量**大都被分配到该函数的栈空间中， 这部分内存由编译器进行管理，编译时确定分配内存的大小。栈空间有特定的结构和寻址方式，所以寻址十分迅速、开销小，只需要2条 CPU 指令，即压栈出栈 `PUSH` 和 `RELEASE`，由于函数栈内存的大小在编译时确定， 所以当局部变量数据太大就会发生栈溢出（`Stack Overflow`）。当函数执行完毕后， 函数的栈空间被回收， 无需手动去释放。

- 堆区
堆空间没有特定的结构，也没有固定的大小，可以动态进行分配和调整，所以内存占用较大的局部变量会放在堆空间上，在编译时不知道该分配多少大小的变量，在运行时也会分配到堆上，在堆上分配内存开销比在栈上大，而且堆上分配的内存需要手动释放，对于 Golang 这种有 GC 机制的语言， 也会增加 GC 压力， 也容易造成内存碎片。

**注：栈是线程级的，堆是进程级的**

#### 二.什么是逃逸分析
逃逸分析在编译阶段确定哪些变量可以分配在栈中，哪些变量分配在堆上。逃逸分析减轻了GC压力，提高程序的运行速度。栈上内存使用完毕不需要GC处理，堆上内存使用完毕会交给GC处理。Go语言使用的是**标记清除算法**，并且在此基础上使用了三色标记法和写屏障技术。

函数传参时对于需要修改原对象值，或占用内存比较大的结构体，选择传指针。对于只读的占用内存较小的结构体，直接传值能够获得更好的执行效率。

根据代码具体分析，尽量减少逃逸代码，减轻GC压力，提高性能。

Go语言中变量不能显示的指定分配在栈空间还是堆空间，但是遵循一个原则：**如果局部变量被其他函数捕获，那么就分配在堆上**。

Go语言的逃逸分析`src/cmd/compile/internal/gc/escape.go`

`pointers to stack objects cannot be stored in the heap`: 指向栈对象的指针不能存储在堆中
`pointers to a stack object cannot outlive that object`: 指向栈对象的指针不能超过该对象的存活期，指针不能在栈对象销毁之后依然存活

逃逸分析是在**编译阶段**进行的，可以通过`go build -gcflga '-m -m l'`查看逃逸分析结果

```sh
go build -gcflags '-m -l' main.go
```
`-m` 会打印出逃逸分析的优化策略，实际上最多总共可以用 4 个 `-m`，但是信息量较大，一般用 1 个就可以了

`-l` 会禁用函数内联，在这里禁用掉 `inline` 能更好的观察逃逸情况，减少干扰。

还可以使用一种更底层的，更硬核，也更准确的方式来判断一个对象是否逃逸： 通过**反编译**命令查看
```sh
go tool compile -S main.go
```

#### 三.逃逸分析案例

1. 函数返回局部指针变量
```go
func Add(x, y int) *int{
  res := 0
  res = x + y
  return &res
}

func main(){
  Add(1, 2)
}
```
```
.\pointer.go:4:2: res escapes to heap:
.\pointer.go:4:2:   flow: ~r2 = &res:
.\pointer.go:4:2:     from &res (address-of) at .\pointer.go:6:9
.\pointer.go:4:2:     from return &res (return) at .\pointer.go:6:2
.\pointer.go:4:2: moved to heap: res
```
函数返回局部变量是一个指针变量，函数Add执行结束，对应栈帧就会销毁，但是指针引用返回到函数外部，如果在函数外部解析使用这个地址，就会导致程序访问非法内存，所以经过编辑器分析过后将其在堆上分配。

2. interface类型逃逸

```go
func main(){
  str := "god"
  fmt.Println(str)
}
```
```
go build -gcflags="-m -m -l" ./pointer.go
# command-line-arguments
.\pointer.go:20:13: str escapes to heap:
.\pointer.go:20:13:   flow: {storage for ... argument} = &{storage for str}:
.\pointer.go:20:13:     from str (spill) at .\pointer.go:20:13
.\pointer.go:20:13:     from ... argument (slice-literal-element) at .\pointer.go:20:13
.\pointer.go:20:13:   flow: {heap} = {storage for ... argument}:
.\pointer.go:20:13:     from ... argument (spill) at .\pointer.go:20:13
.\pointer.go:20:13:     from fmt.Println(... argument...) (call parameter) at .\pointer.go:20:13
.\pointer.go:20:13: ... argument does not escape
.\pointer.go:20:13: str escapes to heap
```
`str`是`main`的一个局部变量，传给 `fmt.Println()`之后逃逸，因为`fmt.Println()`的入参是`interface{}`类型，那么编译期间就很难确定参数类型。

```go
func main(){
  str := "god"
  fmt.Println(&str)
}
```
```
# command-line-arguments
.\pointer.go:19:2: str escapes to heap:
.\pointer.go:19:2:   flow: {storage for ... argument} = &str:
.\pointer.go:19:2:     from &str (address-of) at .\pointer.go:20:14
.\pointer.go:19:2:     from &str (interface-converted) at .\pointer.go:20:14
.\pointer.go:19:2:     from ... argument (slice-literal-element) at .\pointer.go:20:13
.\pointer.go:19:2:   flow: {heap} = {storage for ... argument}:
.\pointer.go:19:2:     from ... argument (spill) at .\pointer.go:20:13
.\pointer.go:19:2:     from fmt.Println(... argument...) (call parameter) at .\pointer.go:20:13
.\pointer.go:19:2: moved to heap: str
.\pointer.go:20:13: ... argument does not escape
```

3. 闭包产生逃逸
```go
func Increase() func() int {
  n := 0
  return func() int {
    n++
    return n
  }
}

func main(){
  in := Increase()
  fmt.Println(in())
}
```
```
go build -gcflags "-m -m -l" ./pointer.go
# command-line-arguments
.\pointer.go:27:2: Increase capturing by ref: n (addr=false assign=true width=8)
.\pointer.go:28:9: func literal escapes to heap:
.\pointer.go:28:9:   flow: ~r0 = &{storage for func literal}:
.\pointer.go:28:9:     from func literal (spill) at .\pointer.go:28:9
.\pointer.go:28:9:     from return func literal (return) at .\pointer.go:28:2
.\pointer.go:27:2: n escapes to heap:
.\pointer.go:27:2:   flow: {storage for func literal} = &n:
.\pointer.go:27:2:     from n (captured by a closure) at .\pointer.go:29:3
.\pointer.go:27:2:     from n (reference) at .\pointer.go:29:3
.\pointer.go:27:2: moved to heap: n
.\pointer.go:28:9: func literal escapes to heap
.\pointer.go:36:16: in() escapes to heap:
.\pointer.go:36:16:   flow: {storage for ... argument} = &{storage for in()}:
.\pointer.go:36:16:     from in() (spill) at .\pointer.go:36:16
.\pointer.go:36:16:     from ... argument (slice-literal-element) at .\pointer.go:36:13
.\pointer.go:36:16:   flow: {heap} = {storage for ... argument}:
.\pointer.go:36:16:     from ... argument (spill) at .\pointer.go:36:13
.\pointer.go:36:16:     from fmt.Println(... argument...) (call parameter) at .\pointer.go:36:13
.\pointer.go:36:13: ... argument does not escape
.\pointer.go:36:16: in() escapes to heap
```
因为**函数是指针类型**，所以匿名函数当做返回值的时候会产生逃逸，匿名函数使用外部变量`n`,这个`n`会一直存在直到`in`方法被销毁。

4. 变量大小不确定及栈空间不足引发逃逸
```go
import (
  "math/rand"
)

func LessThan8192(){
  nums := make([]int, 100) // 64KB
  for i := 0; i <len(nums); i++ {
    nums[i] = rand.Int()
  }
}

func MoreThan8192(){
  nums := make([]int, 1000000) // 64KB
  for i := 0; i <len(nums); i++ {
    nums[i] = rand.Int()
  }
}

func NonConstant(){
  number := 10
  s := make([]int, number)
  for i := 0; i <len(s); i++ {
    s[i] = i
  }
}

func main(){
  NonConstant()
  MoreThan8192()
  LessThan8192()
}
```
```
# command-line-arguments
.\pointer.go:43:14: make([]int, 100) does not escape
.\pointer.go:51:14: make([]int, 1000000) escapes to heap:
.\pointer.go:51:14:   flow: {heap} = &{storage for make([]int, 1000000)}:
.\pointer.go:51:14:     from make([]int, 1000000) (too large for stack) at .\pointer.go:51:14
.\pointer.go:51:14: make([]int, 1000000) escapes to heap
.\pointer.go:60:11: make([]int, number) escapes to heap:
.\pointer.go:60:11:   flow: {heap} = &{storage for make([]int, number)}:
.\pointer.go:60:11:     from make([]int, number) (non-constant size) at .\pointer.go:60:11
.\pointer.go:60:11: make([]int, number) escapes to heap
```
栈空间足够不会发生逃逸，变量过大，已经超过栈空间，会逃逸到堆上

#### 四. 逃逸场景
1. 指针逃逸
在函数中创建了一个对象，返回了这个对象的指针。这种情况下函数虽然退出了，但是因为指针的存在，对象的内存不能随着函数结束而回收，因此只能分配在堆上。
```go
package main

import "fmt"

type Demo struct {
	name string
}

func createDemo(name string) *Demo {
	d := new(Demo) // 局部变量 d 逃逸到堆
	d.name = name
	return d
}

func main() {
	demo := createDemo("demo")
	fmt.Println(demo)
}

```
在这个例子中，函数`createDemo`的局部变量`d`发生了逃逸，`d`作为返回值在`main`函数中继续使用，因此`d`指向的内存不能分配在栈上，只能分配在堆上

2. 动态反射interface{}变量
`interface{}` 可以表示任意的类型，如果函数参数为 `interface{}`，编译期间很难确定其参数的具体类型，也会发生逃逸。
```go
func main(){
  demo := createDemo("demo")
  fmt.Println(demo)
}

// ./main.go:18:13: demo escapes to heap
```
`demo`是`main`函数的一个局部变量，该变量作为实参传递给`fmt.Println()`，因为`fmt.Println()`的参数类型是`interface{}`，因此也发生了逃逸

解释：`fmt.Println` 之类的底层系统函数，实现逻辑会基于`interface{}`做反射，通过 `reflect.TypeOf(arg).Kind()` 获取接口对象的底层数据类型，创建具体类型对象时，会发生内存逃逸。由于 `interface{}` 的变量，编译时无法确定变量类型以及申请空间大小，所以不能在栈空间上申请内存，需要在 `runtime` 时动态申请，理所应当地发生内存逃逸。

3. 申请栈空间过大
栈空间大小是有限的，如果编译时发现局部变量申请的空间过大，则会发生内存逃逸
```go
func main(){
  num := make([]int, 0, 10000)
  _ = num
}

// ./main.go:404:13: make([]int, 0, 10000) escapes to heap
```

经过测试 `num := make([]int, 0, 8193)` 时刚好发生内存逃逸。在 `64` 位机上 `int` 类型为 `8B`，即 `8192 * 8B = 64KB`
```go
func main(){
  num1 := make([]int, 0, 8192)
  _ = num1

  num2 := make([]int, 0, 8193)
  _ = num2
}

// ./main.go:404:14: make([]int, 0, 8192) does net escape
// ./main.go:407:14: make([]int, 0, 8193) escapes to heap
```

4. 切片变量自身和元素的逃逸
未指定`slice`的`len`和`cap`时，`slice`自身未发生逃逸，`slice`的元素发生逃逸。

`slice`动态扩容，编译器不知道容量大小，无法提前在栈空间分配内存，扩容后`slice`的元素可能会被分配到堆空间
```go
type person struct {
  Name string
}

func main(){
  var num []*person
  p1 := &person{
    Name: "Tom",
  }
  num = append(num, p1)
}

// ./main.go:409:8: &person{...} escapes to heap
```

只指定slice的长度即为数组array，数组本身和元素均在栈上分配，均未发生逃逸

