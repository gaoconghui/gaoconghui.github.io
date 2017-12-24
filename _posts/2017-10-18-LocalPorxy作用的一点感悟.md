---
layout: post
title: LocalProxy作用的一点感悟
date: 2017.10.18 21:00 +08:00
tags: 
- python 
- werkzeug
---

这里的LocalProxy是指 werkzeug.local中的一个类

```
class werkzeug.local.LocalProxy(*local*, *name=None*)[]
Acts as a proxy for a werkzeug local. Forwards all operations to a proxied object. The only operations not supported for forwarding are right handed operands and any kind of assignment.
```
用法如下

```
from werkzeug.local import Local
        l = Local()

        # these are proxies
        request = l('request')
        user = l('user')
```

然后就可以像操作一般的对象一般操作代理了。一般的代理模式会做一些小改动，而此处不同，几乎所有的操作会直接作用在代理的对象上。

看到这的时候有个疑问，既然和一般对象一样了，为什么还需要这个类呢？为啥`l('user')`不直接返回user而是返回一个代理呢？

思考许久，才发现自己想到了误区里边。这个包本身就是给协程中使用的，本身的local对象也是每个协程拥有各自的local。返回localproxy，每次调用都会是在该协程的local对象中寻找，如果直接返回对象，则失去了其本身的意义。
