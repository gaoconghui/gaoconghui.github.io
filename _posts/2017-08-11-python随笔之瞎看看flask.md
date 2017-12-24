---
layout: post
title: python随笔之瞎看看flask
date: 2017.08.11 21:00 +08:00
tags: 
 - python随笔
 - flask 
---

### wsgi

在看flask之前，必须要看的是wsgi。

wsgi 更像是一个为了让服务器和应用程序一起工作的标准，遵循标准的application(如flask)可以运行在遵循标准的server（uwsgi等）之上。

这里有一篇实现标准的wsgi server的博客

[自己写一个 wsgi 服务器运行 Django 、Tornado 等框架应用](http://python.jobbole.com/85296/)

### flask

```
app = Flask(__name__)
app.run()
```

##### 上述代码发生了什么？

```python
#flask
def run(self, host='localhost', port=5000, **options):
    """Runs the application on a local development server"""
    from werkzeug import run_simple
    if 'debug' in options:
        self.debug = options.pop('debug')
    if self.static_path is not None:
        options['static_files'] = {
            self.static_path:   (self.package_name, 'static')
        }
    options.setdefault('use_reloader', self.debug)
    options.setdefault('use_debugger', self.debug)
    return run_simple(host, port, self, **options)
```

run 是一个只有在测试的时候才会使用的方法，从werkzeug中import了run_simple ， run_simple 可以认为会运行一个wsgi的server，其中接受的第三个参数即时application。

```python
# werkzeug.serving
def run_simple(hostname, port, application, use_reloader=False,
               use_debugger=False, use_evalex=True,
               extra_files=None, reloader_interval=1,
               reloader_type='auto', threaded=False,
               processes=1, request_handler=None, static_files=None,
               passthrough_errors=False, ssl_context=None):
```

application会接收两个参数

```
def application(environ, start_response)
environ:          一个包含请求信息及环境信息的字典，server 端会详细说明
start_response:   一个接受两个参数`status, response_headers`的方法:
status:           返回状态码，如http 200、404等
response_headers: 返回信息头部列表
```

而上述传入run_simple中的application即为Flask的实例本身，自身作为application，在__call__中接收了这两个参数。

```python
# flask
def __call__(self, environ, start_response):
    """Shortcut for :attr:`wsgi_app`"""
    return self.wsgi_app(environ, start_response)
```

终于，我们找到了核心的方法，wsgi_app，flask处理一个request的流程也能在这比较清晰的看到。flask的具体解析也会在这开始展开。

```python
# flask
def wsgi_app(self, environ, start_response):
    """The actual WSGI application.  This is not implemented in
    `__call__` so that middlewares can be applied:

        app.wsgi_app = MyMiddleware(app.wsgi_app)
    """
    _request_ctx_stack.push(_RequestContext(self, environ))
    try:
        rv = self.preprocess_request()
        if rv is None:
            rv = self.dispatch_request()
        response = self.make_response(rv)
        response = self.process_response(response)
        return response(environ, start_response)
    finally:
        _request_ctx_stack.pop()
```

### flask具体解析

```
_request_ctx_stack.push(_RequestContext(self, environ))
...
_request_ctx_stack.pop()
```

乍一看这两行很迷，但却非常有用。用过flask的都知道，在具体写业务的时候，可以直接import request进来，request中会包含当前请求的一些属性。这个实现就是靠的这两行。

```python
# context locals
_request_ctx_stack = LocalStack()
current_app = LocalProxy(lambda: _request_ctx_stack.top.app)
request = LocalProxy(lambda: _request_ctx_stack.top.request)
session = LocalProxy(lambda: _request_ctx_stack.top.session)
g = LocalProxy(lambda: _request_ctx_stack.top.g)
```

可以看到，_request_ctx_stack就是一个LocalStack对象，LocalStack是werkzeug中的一个对象，可以简单的认为每一个线程，或者是每一个协程在localStack中存储的对象都会相互独立。具体关于LocalStack相关可以看http://python.jobbole.com/87738/。
对于每个请求，flask都会将其压入栈顶，方便后边使用，全部处理结束后再弹出。

```
rv = self.preprocess_request()
if rv is None:
    rv = self.dispatch_request()
response = self.make_response(rv)
response = self.process_response(response)
return response(environ, start_response)
```

再看中间的几行，看上去像是几行非常简单的模板方法模式，先preprocess，然后dispatch，再生成response，最后再处理一下response。
preprocess_request 和process_response两个方法非常的相似，都是类似责任链的实现，以preprocess_request为例

```python
for func in self.request_init_funcs:
    rv = func()
    if rv is not None:
        return rv
```

request_init_funcs是一个list，会通过request_init这个装饰器添加

```python
def request_init(self, f):
    """Registers a function to run before each request."""
    self.request_init_funcs.append(f)
    return f
```

接着看dispatch_request

```python
def dispatch_request(self):
    try:
        endpoint, values = self.match_request()
        return self.view_functions[endpoint](**values)
    except HTTPException, e:
        handler = self.error_handlers.get(e.code)
        if handler is None:
            return e
        return handler(e)
    except Exception, e:
        handler = self.error_handlers.get(500)
        if self.debug or handler is None:
            raise
        return handler(e)
```

顺便列出match_request 以及_RequestContext

```python
def match_request(self):
    rv = _request_ctx_stack.top.url_adapter.match()
    request.endpoint, request.view_args = rv
    return rv
```

```python
class _RequestContext(object):
    def __init__(self, app, environ):
        self.app = app
        self.url_adapter = app.url_map.bind_to_environ(environ)
        self.request = app.request_class(environ)
        self.session = app.open_session(self.request)
        self.g = _RequestGlobals()
        self.flashes = None
```

dispatch_request 调用match_request, match_request调用urladapter, urladapter是由app.url_map生成的，url_map 是个Map对象（在Flask __init__中申明），而这个Map对象是werkzeug.routing中的对象，也就是说，flask的route部分由werkzeug实现。这部分比较复杂，之后单独写一个文章来讲werkzeug.routing模块，这边先略过，只用知道通过route装饰器会将rule跟具体方法做一个映射，并且在需要的时候能通过match_request方法找到即可。

```python
def route(self, rule, **options):
    """A decorator that is used to register a view function for a
    given URL rule.  Example::

        @app.route('/')
        def index():
            return 'Hello World'
    """
    def decorator(f):
        if 'endpoint' not in options:
            options['endpoint'] = f.__name__
        self.url_map.add(Rule(rule, **options))
        self.view_functions[options['endpoint']] = f
        return f
    return decorator
```

而后会对业务代码返回的response用make_response包装，返回一个response 对象（没错，又是一个werkzeug中的对象)。

### 小结

以上都是基于flask最早的那个commit分析的，最早的版本flask代码量相当少，重要部分基本都是直接使用werkzeug。而且flask的大部分特性在这个版本中都没体现。。