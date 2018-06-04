---
layout: post
title: 对go中function type的一点思考
description: 对go中function type的一点思考
date: 2018.05.30 21:00 +08:00
tags: 
- go
---

function type 可以理解为一组拥有相同参数类型和结果类型的方法的集合。我看也有人管他叫接口型函数。

>A function type denotes the set of all functions with the same parameter and result types. The value of an uninitialized variable of function type is `nil`.

> Two function types are identical if they have the same number of parameters and result values, corresponding parameter and result types are identical, and either both functions are variadic or neither is. Parameter and result names are not required to match.

下面是function type最一般的打开方式

```go
type Handler func(string) string

func process1(h Handler, s string) string {
	return "process1" + h(s)
}

func process2(h Handler, s string) string {
	return "process2" + h(s)
}

```

我们顶一个一个Handler，输入参数为string，输出为string，且有一系列的复杂方法都以Handler作为输入参数，做一些处理。这种情况下，我们可以很方便的去复用process1和process2方法。

```go
processResult := process1(func(s string) string {
	return strings.ToLower(s)
}, "gch")

```

process1的第一个参数为Handler，我们只需要构建一个和它拥有同样参数以及返回类型的方法即可。

接下来是一种更神奇的用法，我们知道，方法可以被声明到任意类型（只要不是一个指针或者一个interface），因此，我们可以给function type添加一个方法。

```go
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers. If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

这个例子来源于`net/http`包，我们只需要定义一个方法，拥有相同的参数，如

```go
func NotFound(w ResponseWriter, r *Request) { Error(w, "404 page not found", StatusNotFound) }
```

就能将这个方法转型为HandlerFunc，并调用ServerHTTP方法。

个人觉得这种给function附带一个function的方法可读性不太强，第一次看到会觉得很奇怪，更常见的用法是使用struct和interface组合起来使用，能以差不多的行的代码实现相同的功能。

```go
type HandleS struct {
	handle func(w http.ResponseWriter, r *http.Request)
}

func NewHandler(f func(w http.ResponseWriter, r *http.Request)) Handlers {
	return &HandlerS{handle: f}
}

func (h *HandlerS) ServerHTTP(w http.ResponseWriter, r *http.Request) {
	h.Handle(w, r)
}
```

也许道行不深，以后会有新的想法。