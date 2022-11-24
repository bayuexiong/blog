---
title: "Go http Server 框架的简单实现"
date: "2022-11-16"
---

在Go想使用 http server，最简单的方法是使用 http/net

```go
err := http.ListenAndServe(":8080", nil)
if err != nil {
    panic(err.Error())
}
http.HandleFunc("/hello", func(writer http.ResponseWriter, request *http.Request) {
    writer.Write([]byte("Hello"))
})
```

定义 handle func

`type HandlerFunc func(ResponseWriter, *Request)`

标准库的 http 服务器实现很简单，开启一个端口，注册一个实现`HandlerFunc`的 handler

同时标准库也提供了一个完全接管请求的方法

```go
func main() {
	err := http.ListenAndServe(":8080", &Engine{})
	if err != nil {
		panic(err.Error())
	}
}

type Engine struct {
}

func (e *Engine) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if r.URL.Path == "/hello" {
		w.Write([]byte("Hello"))
	}
}
```

定义 ServerHTTP

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

如果我们需要写一个 HTTP Server 框架，那么就需要实现这个方法，同时 net/http 的输入输出流都不是很方便，我们也需要包装，再加上一个简单的 Route，不要在 ServeHTTP 里面写Path。

这里稍微总结一下

1. 一个实现 ServeHTTP 的Engine

2. 一个包装 HTTP 原始输入输出流的 Context

3. 一个实现路由匹配的 Route

Route 这边为了简单，使用Map做完全匹配

```go
import (
	"net/http"
)

type Engine struct {
	addr  string
	route map[string]handFunc
}

type Context struct {
	w      http.ResponseWriter
	r      *http.Request
	handle handFunc
}

type handFunc func(ctx *Context) error

func NewServer(addr string) *Engine {
	return &Engine{
		addr:  addr,
		route: make(map[string]handFunc),
	}
}

func (e *Engine) Run() {
	err := http.ListenAndServe(e.addr, e)
	if err != nil {
		panic(err)
	}
}

func (e *Engine) Get(path string, handle handFunc) {
	e.route[path] = handle
}

func (e *Engine) handle(writer http.ResponseWriter, request *http.Request, handle handFunc) {
	ctx := &Context{
		w:      writer,
		r:      request,
		handle: handle,
	}
	ctx.Next()
}

func (e *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	handleF := e.route[req.URL.Path]
	e.handle(w, req, handleF)
}

func (c *Context) Next() error {
	return c.handle(c)
}

func (c *Context) Write(s string) error {
	_, err := c.w.Write([]byte(s))
	return err
}
```


我们写一个Test验证一下我们的 Http Server

```go
func TestHttp(t *testing.T) {
	app := NewServer(":8080")

	app.Get("/hello", func(ctx *Context) error {
		return ctx.Write("Hello")
	})

	app.Run()
}
```

这边我们包装的 Handle 使用了是 Return error 模式，相比标准库只 Write 不 Return ，避免了不 Write 之后忘记 Return 导致的错误，这通常很难发现。

一个Http Server 还需要一个 middleware 功能，这里的思路就是在 Engine 中存放一个 handleFunc 的数组，支持在外部注册，当一个请求打过来时创建一个新 Ctx，将 Engine 中全局的 HandleFunc 复制到 Ctx 中，再使用 c.Next() 实现套娃式调用。

```go
package http

import (
	"net/http"
)

type Engine struct {
	addr        string
	route       map[string]handFunc
	middlewares []handFunc
}

type Context struct {
	w        http.ResponseWriter
	r        *http.Request
	index    int
	handlers []handFunc
}

type handFunc func(ctx *Context) error

func NewServer(addr string) *Engine {
	return &Engine{
		addr:        addr,
		route:       make(map[string]handFunc),
		middlewares: make([]handFunc, 0),
	}
}

func (e *Engine) Run() {
	err := http.ListenAndServe(e.addr, e)
	if err != nil {
		panic(err)
	}
}

func (e *Engine) Use(middleware handFunc) {
	e.middlewares = append(e.middlewares, middleware)
}

func (e *Engine) Get(path string, handle handFunc) {
	e.route[path] = handle
}

func (e *Engine) handle(writer http.ResponseWriter, request *http.Request, handle handFunc) {
	handlers := make([]handFunc, 0, len(e.middlewares)+1)
	handlers = append(handlers, e.middlewares...)
	handlers = append(handlers, handle)
	ctx := &Context{
		w:        writer,
		r:        request,
		index:    -1,
		handlers: handlers,
	}
	ctx.Next()
}

func (e *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	handleF := e.route[req.URL.Path]
	e.handle(w, req, handleF)
}

func (c *Context) Next() error {
	c.index++
	if c.index < len(c.handlers) {
		return c.handlers[c.index](c)
	}
	return nil
}

func (c *Context) Write(s string) error {
	_, err := c.w.Write([]byte(s))
	return err
}
```

实现方法很简单，这里我们验证一下是否可以支持前置和后置中间件

```go
func TestHttp(t *testing.T) {
	app := NewServer(":8080")

	app.Get("/hello", func(ctx *Context) error {
		fmt.Println("Hello")
		return ctx.Write("Hello")
	})
	app.Use(func(ctx *Context) error {
		fmt.Println("A1")
		return ctx.Next()
	})
	app.Use(func(ctx *Context) error {
		err := ctx.Next()
		fmt.Println("B1")
		return err
	})

	app.Run()

}
```

输出：

```go
=== RUN   TestHttp
A1
Hello
B1
```

一共不过100行代码，我们就实现了一个简易的 Http Server ，后续细节可以去看Gin的源码，主要关注点在 Route 方面，前缀树的实现。