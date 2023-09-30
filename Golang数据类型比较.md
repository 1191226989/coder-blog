---
title: Golang数据类型比较
created: '2023-09-30T02:17:13.653Z'
modified: '2023-09-30T05:29:36.007Z'
---

# Golang数据类型比较

| 分类 | 说明 |	是否可比较 | 说明 |
| --- | --- | --- | --- |
| 基本类型 | 整型/浮点数/复数类型/字符串 | 是 | |	
| 引用类型 | 切片（slice）/map | 否	| panic: runtime error: comparing uncomparable type []int |
| 复合类型 | 数组 |	是 | 相同长度的数组可以比较，不同长度的数组不能进行比较
| 复合类型 | 结构体 | 是 | 只包含可比较的类型情况下可比较 |
| 接口类型 | error |	是 | |

#### 基本类型
1. 浮点数
```go
package main 

import "fmt" 

func main() {     
    var a float64=0.1     
    var b float64=0.2     
    fmt.Println(a==b) // true
    fmt.Println(a+b) // 0.30000000000000004  
}

```
2. 字符串
```go
package golangbase

import (
	"fmt"
	"testing"
)

func TestString(t *testing.T) {
	str1 := "哈哈"
	str2 := "哈哈"
	fmt.Println(str1 == str2) // true
}

```
#### 引用类型
1. slice、map
> 切片之间不允许比较。切片只能与nil值比较
> map之间不允许比较。map只能与nil值比较
> 两个nil也不能比较，会panic

使用reflect.DeepEqual()对比规则
- 相同类型的值是深度相等的，不同类型的值永远不会深度相等。
- 当数组值（array）的对应元素深度相等时，数组值是深度相等的。
- 当结构体（struct）值如果其对应的字段（包括导出和未导出的字段）都是深度相等的，则该值是深度相等的。
- 当函数（func）值如果都是零，则是深度相等；否则就不是深度相等。
- 当接口（interface）值如果持有深度相等的具体值，则深度相等。
- 当切片（slice）序号相同，如果值和指针都相等，那么就是深度相等的
- 当哈希表（map）相同的key，如果值和指针都相等，那么就是深度相等的。

```go
package golangbase

import (
	"reflect"
	"testing"
)

type StructA struct {
	Name  string
	Hobby []string
}

type StructB struct {
	Name string
}

func TestDeepEqual(t *testing.T) {
	s1 := StructA{Name: "test", Hobby: []string{"唱", "跳"}}
	s2 := StructA{Name: "test", Hobby: []string{"唱", "跳"}}
	println(reflect.DeepEqual(s1, s2))// true
	mp1 := map[int]int{1: 10, 2: 20}
	mp2 := map[int]int{1: 10, 2: 20}
	println(reflect.DeepEqual(mp1, mp2))// true
}

```
2. channel、指针
- 指针可比较，只要指针指向的地址一样，则相等
- 由于通过make创建channel后，返回的是一个指针，所以可以比较

```go
c1 := make(chan int, 2) 
c2 := make(chan int, 2) 
c3 := c1 
fmt.Println(c3 == c1) // true 
fmt.Println(c2 == c1) // false 
```

#### 复合类型
1. 数组
相同长度的数组是可以比较的，而不同长度的数组是不能进行比较的。因为数组类型中,数组的长度也是类型的一部分，不同长度的数组就被认为不同的类型，所以无法比较
```go
package main

import "fmt"

func main() {
    a := [2]int{1, 2}
    b := [2]int{1, 2}
    c := [2]int{1, 3}
    d := [3]int{1, 2, 4}
    fmt.Println(a == b) // true
    fmt.Println(a == c) // false
    fmt.Println(a == d) // invalid operation: a == d (mismatched types [2]int and [3]int)
}

```
2. 结构体
结构体所有字段的类型全部为可比较的类型
```go
package main
import "fmt"

type A struct {
    id int
    name string
}
func main() {
    a := A{id:5,name:"123"}
    b := A{id:5,name:"123"}
    c := A{id:5,name:"1234"}
    fmt.Println(a == b) // true
    fmt.Println(a == c) // false
}

```
slice不可比较，如果结构体包含了slice，则不可比较
```go
package main
import "fmt"

type A struct {
    id int
    name string
    son []int
}
func main() {
    a := A{id:5,name:"123",son:[]int{1,2,3}}
    b := A{id:5,name:"123",son:[]int{1,2,3}}
    fmt.Println(a == b) // invalid operation: a == b (struct containing []int cannot be compared)
}

```
3. 接口
根据接口类型是否包含一组方法将接口类型分成了两类：
- 使用 `runtime.iface` 结构体表示包含方法的接口
- 使用 `runtime.eface` 结构体表示不包含任何方法的 interface{} 类型
```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}

type iface struct {
    tab  *itab
    data unsafe.Pointer
}

```
一个接口值是由两个部分组成的，即该接口对应的**类型**和接口对应**具体的值**。只有当类型和值都相等（动态值使用==比较），两个接口值才是相等的

```go
var a interface{} = 0
var b interface{} = 2
var c interface{} = 0
var d interface{} = 0.0
fmt.Println(a == b) // false
fmt.Println(a == c) // true
fmt.Println(a == d) // false

```

