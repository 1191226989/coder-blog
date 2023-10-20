---
title: Golang设计模式-装饰器模式
created: '2023-10-06T07:13:35.954Z'
modified: '2023-10-20T06:55:42.868Z'
---

# Golang设计模式-装饰器模式

#### 概念：
一种动态地往一个类（结构体）添加新的行为的设计模式

装饰器模式相对于生成子类（继承）更为灵活，这样可以给某个对象而不是整个结构体都添加一些功能

它是一种对象结构型模式

#### 优点：
可以通过一种动态的方式来扩展一个对象（结构体）的功能
    
可以使用多个具体的装饰器来装饰同一个结构体，增加其功能

具体组件类与具体装饰器可以独立变化，符合 “开闭原则”

#### 缺点：
对于多次装饰的结构体，容易出错，排查错误也很困难

增加系统的复杂度以及理解成本

#### 适合场景：
需要给一个结构体增加功能，这些功能可以动态的撤销

需要给一批兄弟类增加或者改装功能

#### 代码示例：
```go
component.go

package component

import (
	"fmt"
)

// 定义一个抽象的组件
type Component interface {
	Operate()
}

// 实现一个具体的组件1
type Component1 struct {
}

func (c1 Component1) Operate() {
	fmt.Println("c1 Operate")
}

// 定义一个抽象的装饰器
type Decorator interface {
	Component
	Do() // 装饰器额外附件的方法
}

// 实现一个具体的装饰器
type Dog struct {
	c Component
}

func (d *Dog) Do() {
	fmt.Println("Dog decorator Do()")
}

func (d *Dog) Operate() {
	fmt.Println("Dog component Operate()")
}

// 实现一个具体的装饰器
type Cat struct {
	c Component
}

func (d *Cat) Do() {
	fmt.Println("Cat decorator Do()")
}

func (d *Cat) Operate() {
	fmt.Println("Cat component Operate()")
}
```

component_test.go
```go
package component

import (
	"testing"
)

func TestComponent(t *testing.T) {
	c1 := &Component1{}
	c1.Operate()
}

func TestDecorator(t *testing.T) {
	d := &Dog{}
	d.c = &Component1{}
	d.Operate()
	d.Do()
	d.c.Operate()
}
```

中间件：
```go
package main

import (
	"log"
	"net/http"
	"time"
)

// 实现一个http server
// 实现一个handler hello
// 实现一个中间件功能 1.记录请求的URL和方法 2.记录请求的网络地址 3.记录方法的执行时间

func tracing(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("tracing start")
		log.Printf("记录请求的URL和方法: %s %s", r.URL, r.Method)
		next.ServeHTTP(w, r)
		log.Println("tracing end")
	})
}

func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("logging start")
		log.Printf("记录请求的网络地址: %s", r.RemoteAddr)
		next.ServeHTTP(w, r)
		log.Println("logging end")
	})
}

func timing(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("timing start")
		startTime := time.Now()
		next.ServeHTTP(w, r)
		elapsedTime := time.Since(startTime)
		log.Printf("记录方法的执行时间: %s", elapsedTime)
		log.Println("timing end")
	})
}

func hello(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello"))
}

func main() {
	http.Handle("/", tracing(logging(timing(http.HandlerFunc(hello)))))
	http.ListenAndServe(":8080", nil)
}
```
```
2022/02/25 18:44:58 tracing start
2022/02/25 18:44:58 记录请求的URL和方法: / GET
2022/02/25 18:44:58 logging start
2022/02/25 18:44:58 记录请求的网络地址: 127.0.0.1:50491
2022/02/25 18:44:58 timing start
2022/02/25 18:44:58 记录方法的执行时间: 0s
2022/02/25 18:44:58 timing end
2022/02/25 18:44:58 logging end
2022/02/25 18:44:58 tracing end
```

