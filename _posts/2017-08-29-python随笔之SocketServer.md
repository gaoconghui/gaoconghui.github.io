---
layout: post
title: python随笔之SocketServer
date: 2017.08.29 21:00 +08:00
tags: 
 - python随笔 
---

#### 从SocketServer 讲起

        +------------+
        | BaseServer |
        +------------+
              |
              v
        +-----------+        +------------------+
        | TCPServer |------->| UnixStreamServer |
        +-----------+        +------------------+
              |
              v
        +-----------+        +--------------------+
        | UDPServer |------->| UnixDatagramServer |
        +-----------+        +--------------------+

整个模块的结构大概是这样的，由BaseServer衍生出TCPServer，UDPServer，然后以此为基类，后续还有HTTPServer(SocketServer.TCPServer)，class WSGIServer(HTTPServer)，但入口都是接下去要说的SockerServer中的一些类。

BaseServer 有以下的一些方法

```
Methods for the caller:

    - __init__(server_address, RequestHandlerClass)
    - serve_forever(poll_interval=0.5)
    - shutdown()
    - handle_request()  # if you do not use serve_forever()
    - fileno() -> int   # for select()

    Methods that may be overridden:

    - server_bind()
    - server_activate()
    - get_request() -> request, client_address
    - handle_timeout()
    - verify_request(request, client_address)
    - server_close()
    - process_request(request, client_address)
    - shutdown_request(request)
    - close_request(request)
    - handle_error()

    Methods for derived classes:

    - finish_request(request, client_address)

    Class variables that may be overridden by derived classes or
    instances:

    - timeout
    - address_family
    - socket_type
    - allow_reuse_address

    Instance variables:

    - RequestHandlerClass
    - socket
```

这其中有一系列钩子方法留着给子类复写以实现其特有的功能。其中实现具体处理逻辑的为_handle_request_noblock方法


```python
def _handle_request_noblock(self):
    """Handle one request, without blocking.

    I assume that select.select has returned that the socket is
    readable before this function was called, so there should be
    no risk of blocking in get_request().
    """
    try:
        request, client_address = self.get_request()
    except socket.error:
        return
    if self.verify_request(request, client_address):
        try:
            self.process_request(request, client_address)
        except:
            self.handle_error(request, client_address)
            self.shutdown_request(request)
```

get\_request 需要子类来实现，获取request和address，而后由process\_request 来处理，process\_request 会依次调用finish\_request 和shutdown\_request,finish_request会调用RequestHandlerClass（需要传入，为具体处理的handler）。

```python
def serve_forever(self, poll_interval=0.5):
    self.__is_shut_down.clear()
    try:
        while not self.__shutdown_request:
            # XXX: Consider using another file descriptor or
            # connecting to the socket to wake this up instead of
            # polling. Polling reduces our responsiveness to a
            # shutdown request and wastes cpu at all other times.
            r, w, e = _eintr_retry(select.select, [self], [], [],
                                   poll_interval)
            if self in r:
                self._handle_request_noblock()
    finally:
        self.__shutdown_request = False
        self.__is_shut_down.set()
```

在serve\_forever 中，会使用select 获取需要处理的内容，再调用上边的\_handle\_request\_noblock。

简单的流程是这样的，BaseServer中定义了一些处理流程方面的东西，并没有任何具体的处理，所有的处理流程，都留给了子类。另外还有些上述没提到的留给TCP 和UDPserver的方法，比如TCP的server\_bind等不一一说了，具体看代码。

#### Threading , Forking
这个模块非常巧妙的使用mix-in class实现了Threading和Forking。

通过上面的分析我们知道，核心的处理逻辑是process\_request方法，以多线程为例，如果我们要实现多线程，其实只需要对这个方法做些改动即可。

```python
def process_request_thread(self, request, client_address):
    """Same as in BaseServer but as a thread.
    In addition, exception handling is done here.
    """
    try:
        self.finish_request(request, client_address)
        self.shutdown_request(request)
    except:
        self.handle_error(request, client_address)
        self.shutdown_request(request)

def process_request(self, request, client_address):
    """Start a new thread to process the request."""
    t = threading.Thread(target = self.process_request_thread,
                         args = (request, client_address))
    t.daemon = self.daemon_threads
    t.start()
```

```
class ThreadingUDPServer(ThreadingMixIn, UDPServer): pass
```

就是这么简单的实现，同样的还有ForkingMixIn ， 这两个mix-in class在后续很多地方都还会使用到，简直是mix-in使用的完美范例。

#### handler
handler的基类是BaseRequestHandler,非常的短。

```python
def __init__(self, request, client_address, server):
    self.request = request
    self.client_address = client_address
    self.server = server
    self.setup()
    try:
        self.handle()
    finally:
        self.finish()

def setup(self):
    pass

def handle(self):
    pass

def finish(self):
    pass
```

而后还有两个具体的实现，StreamRequestHandler 和DatagramRequestHandler，分别用于TCP 和UDPServer中，也只是简单实现了setup和finish，核心handle方法留给子类实现。

如BaseHttpServer中，有一个BaseHTTPRequestHandler，针对http协议做了一系列处理，当然，既然以base开头，也只是为后续实现提供方便。如werkzeug.serving中的WSGIRequestHandler，在继承BaseHTTPRequestHandler的基础上，做了一些修改，用于适配wsgi标准。如需要传入application，用于处理具体业务。
