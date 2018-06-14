---
layout: post
title: golang hijack打开方式
description: golang http包中hijack方法使用的见解
date: 2018.06.13 21:00 +08:00
tags: 
- go
---



### 简介Hijack

```go
type Hijacker interface {
	// Hijack lets the caller take over the connection.
	// After a call to Hijack the HTTP server library
	// will not do anything else with the connection.
	//
	// It becomes the caller's responsibility to manage
	// and close the connection.
	//
	// The returned net.Conn may have read or write deadlines
	// already set, depending on the configuration of the
	// Server. It is the caller's responsibility to set
	// or clear those deadlines as needed.
	//
	// The returned bufio.Reader may contain unprocessed buffered
	// data from the client.
	//
	// After a call to Hijack, the original Request.Body must
	// not be used.
	Hijack() (net.Conn, *bufio.ReadWriter, error)
}
```

`Hijack()`可以将HTTP对应的TCP连接取出，连接在`Hijack()`之后，HTTP的相关操作就会受到影响，调用方需要负责去关闭连接。看一个简单的例子。

```go
func handle1(w http.ResponseWriter, r *http.Request) {
	hj, _ := w.(http.Hijacker)
	conn, buf, _ := hj.Hijack()
	defer conn.Close()
	buf.WriteString("hello world")
	buf.Flush()
}

func handle2(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "hello world")
}
```

问题来了，上面两个`handle`方法有什么区别呢？很简单，同样是http请求，返回的结果一个遵循http协议，一个不遵循。

```
➜  ~ curl -i http://localhost:9090/handle1
hello world%                                                                                                                                                                                                                            ➜  ~ curl -i http://localhost:9090/handle2
HTTP/1.1 200 OK
Date: Thu, 14 Jun 2018 07:51:31 GMT
Content-Length: 11
Content-Type: text/plain; charset=utf-8

hello world%
```

分别是以上两者的返回，可以看到，hijack之后的返回，虽然body是相同的，但是完全没有遵循http协议。（废话，别人都说了hijack之后返回了body然后直接关闭了，哪来的headers = = ）

但我们还是要看看为啥..

```go
func (c *conn) serve(ctx context.Context) {
	...
  	serverHandler{c.server}.ServeHTTP(w, w.req)
    w.cancelCtx()
    if c.hijacked() {
      return
    }
    w.finishRequest()
  	...
}
```

这是net/http包中的方法，也是http路由的核心方法。调用`ServeHTTP`（也就是上边的handle方法）方法，如果被`hijack`了就直接return了，而一般的http请求会经过后边的`finishRequest`方法，加入headers等并关闭连接。

### 打开方式

上边我们说了`Hijack`方法，一般在在创建连接阶段使用HTTP连接，后续自己完全处理connection。符合这样的使用场景的并不多，基于HTTP协议的rpc算一个，从HTTP升级到WebSocket也算一个。

#### RPC中的应用

go中自带的rpc可以直接复用http server处理请求的那一套流程去创建连接，连接创建完毕后再使用`Hijack`方法拿到连接。

```go
// ServeHTTP implements an http.Handler that answers RPC requests.
func (server *server) servehttp(w http.responsewriter, req *http.request) {
	if req.method != "connect" {
		w.header().set("content-type", "text/plain; charset=utf-8")
		w.writeheader(http.statusmethodnotallowed)
		io.writestring(w, "405 must connect\n")
		return
	}
	conn, _, err := w.(http.hijacker).hijack()
	if err != nil {
		log.print("rpc hijacking ", req.remoteaddr, ": ", err.error())
		return
	}
	io.writestring(conn, "http/1.0 "+connected+"\n\n")
	server.serveconn(conn)
}
```

客户端通过向服务端发送method为connect的请求创建连接，创建成功后即可开始rpc调用。

#### websocket中的应用

```go
// ServeHTTP implements the http.Handler interface for a WebSocket
func (s Server) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	s.serveWebSocket(w, req)
}

func (s Server) serveWebSocket(w http.ResponseWriter, req *http.Request) {
	rwc, buf, err := w.(http.Hijacker).Hijack()
	if err != nil {
		panic("Hijack failed: " + err.Error())
	}
	// The server should abort the WebSocket connection if it finds
	// the client did not send a handshake that matches with protocol
	// specification.
	defer rwc.Close()
	conn, err := newServerConn(rwc, buf, req, &s.Config, s.Handshake)
	if err != nil {
		return
	}
	if conn == nil {
		panic("unexpected nil conn")
	}
	s.Handler(conn)
}
```

websocket在创建连接的阶段与http使用相同的协议，而在后边的数据传输的过程中使用了他自己的协议，符合了`Hijack`的用途。通过`serveWebSocket`方法将HTTP协议升级到Websocket协议。