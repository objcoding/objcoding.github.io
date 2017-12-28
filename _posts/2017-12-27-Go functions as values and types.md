---
layout: post
title: "Go函数作为值与类型"
categories: Go
tags: function
author: zch
---

* content
{:toc}

在 Go 语言中，我们可以把函数作为一种变量，用 type 去定义它，那么这个函数类型就可以作为值传递，甚至可以实现方法，这一特性是在太灵活了，有时候我们甚至可以利用这一特性进行类型转换。作为值传递的条件是类型具有相同的参数以及相同的返回值。









## 函数的类型转换

Go 语言的类型转换基本格式如下：

```
type_name(expression)
```

举个例子：

```go
package main

import "fmt"

type CalculateType func(int, int) // 声明了一个函数类型

// 该函数类型实现了一个方法
func (c *CalculateType) Serve() {
  fmt.Println("我是一个函数类型")
}

// 加法函数
func add(a, b int) {
  fmt.Println(a + b)
}

// 乘法函数
func mul(a, b int) {
  fmt.Println(a * b)
}

func main() {
  a := CalculateType(add) // 将add函数强制转换成CalculateType类型
  b := CalculateType(mul) // 将mul函数强制转换成CalculateType类型
  a(2, 3)
  b(2, 3)
  a.Serve()
  b.Serve()
}

// 5
// 6
// 我是一个函数类型
// 我是一个函数类型
```

如上，声明了一个 CalculateType 函数类型，并实现 Serve() 方法，并将拥有相同参数的 add 和 mul 强制转换成 CalculateType 函数类型，同时这两个函数都拥有了 CalculateType 函数类型的 Serve() 方法。



## 函数作参数传递

```go
package main

import "fmt"

type CalculateType func(a, b int) int // 声明了一个函数类型

// 加法函数
func add(a, b int) int {
  return a + b
}

// 乘法函数
func mul(a, b int) int {
  return a * b
}

func Calculate(a, b int, f CalculateType) int {
  return f(a, b)
}

func main() {
  a, b := 2, 3
  fmt.Println(Calculate(a, b, add))
  fmt.Println(Calculate(a, b, mul))
}
// 5
// 6
```

如上例子，Calculate 的 f 参数类型为 CalculateType，add 和 mul 函数具有和 CalculateType 函数类型相同的参数和返回值，因此可以将 add 和 mul 函数作为参数传入 Calculate 函数中。





## net/http 包源码例子



```go
// HandleFunc registers the handler function for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
  DefaultServeMux.HandleFunc(pattern, handler)
}
```



```go
// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
  mux.Handle(pattern, HandlerFunc(handler))
}
```



```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
  f(w, r)
}
```

刚开始看到这段源码的时候，真的有点懵逼了，这段源码的目的是为了将我们的 Handler 强制实现 ServeHTTP() 方法，如下例子：

```go
func sayHi(w http.ResponseWriter, r *http.Request) {
  io.WriteString(w, "hi")
}

func main() {
  http.HandlerFunc("/", sayHi)
  http.ListenAndserve(":8080", nil)
}
```

因为 HandlerFunc 是一个函数类型，而 sayHi 函数拥有和 HandlerFunc 函数类型一样的参数值，因此可以将 sayHi 强制转换成 HandlerFunc，因此 sayHi 也拥有了 ServeHTTP() 方法，也就实现了 Handler 接口，同时，HandlerFunc 的 ServeHTTP 方法执行了它自己本身，也就是 sayHi 函数，这也就可以看出来了，sayHi 就是 Handler 被调用之后的执行结果。
