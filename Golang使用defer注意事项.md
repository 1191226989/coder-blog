---
title: Golang使用defer注意事项
created: '2023-09-30T01:26:04.426Z'
modified: '2023-09-30T02:09:01.429Z'
---

# Golang使用defer注意事项

在 golang 当中 defer 代码块会在**函数调用链表**中增加一个函数调用。这个函数调用不是普通的函数调用，而是会在函数正常返回，也就是 return 之后添加一个函数调用。因此 defer 通常用来释放函数内部变量。

#### 规则 1 当 defer 被声明时，其参数就会被实时解析
```go
func a() {
	i := 0
	defer fmt.Println(i)
	i++
	return
}
```
结果输出的是 0

在 defer 后面定义的是一个带变量的函数: fmt.Println (i). 但这个变量 (i) 在 defer 被声明的时候，就已经确定其确定的值了。 换言之，上面的代码等同于下面的代码:
```go
func a() {
	i := 0
	defer fmt.Println(0) //因为i=0，所以此时就明确告诉golang在程序退出时，执行输出0的操作
	i++
	return
}
```
通过运行结果，可以看到 defer 输出的值，就是定义时的值，而不是 defer 真正执行时的变量值。

#### 规则 2 defer 执行顺序为先进后出
当同时定义了多个 defer 代码块时，golang 会按照先定义后执行的顺序依次调用 defer，可以理解为一个 defer 函数栈（先进后出）。

#### 规则 3 defer 可以读取有名返回值
```go
func c() (i int) {
	defer func() { 
		i++ 
	}()
	return 1
}
```
输出结果是 2

函数 defer 是在 return 调用之后才执行的。 这里需要明确的是 defer 代码块的作用域仍然在函数之内，结合上面的函数也就是说 defer 的作用域仍然在 c 函数之内。因此 defer 仍然可以读取 c 函数内的变量 (如果无法读取函数内变量，那又如何进行变量清除呢...)。

当执行 return 1 之后，i 的值就是 1。此时此刻，defer 代码块开始执行，对 i 进行自增操作， 因此输出 2。

