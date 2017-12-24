---
layout: post
title: python随笔之实现classmethod staticmethod
date: 2017.11.20 21:00 +08:00
tags: 
 - python随笔 
---

大概刚学python的时候就知道了classmethod和staticmethod的用法，后来看到描述器才知道描述器与他们之间千丝万缕的关系，是否能动手实现下这俩货嘞？这边先从bound和unbound method说起

#### bound unbound

看下面的例子

```python
class Cla():
    def __init__(self,a):
        self.a = a
    def f(self):
        print self,self.a
        
print Cla.f
# <unbound method Cla.f>
print Cla(1).f
# <bound method Cla.f of <__main__.Cla instance at 0x103f3fb00>>

```

这个非常好理解，bound指的是是否和具体实例绑定，我们知道，调用实例的方法，第一个参数是self，需要给第一个参数传入一个对象。

```python
print Cla(1).f()
print Cla.f()
print Cla.f(Cla(2))
```

第一个和第三个能成功执行，第二个会报错。报错为`unbound method f() must be called with Cla instance as first argument (got nothing instead)`

原因如下：

`Cla.f` 相当于`Cla.__dict__['f'].__get__(None,Cla)`，返回的是unbound method

`Cla(1).f` 相当于`Cla.__dict__['f'].__get__(Cla(1),Cla)`，返回的是bound method

两者都会返回一个函数，不同的是，bound method会把Cla(1)绑定到f的第一个参数上，但是unbound method不会，不过可以传入一个参数（这个参数必须是Cla的实例）

大致相当于下面，但并不准确，只是大概实现。需要注意的是，下边全部都是新式类，经典类会表现的不太一样。（不太想研究经典类这种过时的东西了...）

```python
class UnboundMethod(object):
    def __init__(self, f, own):
        self.f = f
        self.own = own

    def __call__(self, *args, **kwargs):
        return self.f(*args, **kwargs)

    def __str__(self):
        return '<unbound method %s.%s>' % (self.own.__name__, self.f.__name__)


class BoundMethod(object):
    def __init__(self, f, obj, own):
        self.f = f
        self.obj = obj
        self.own = own

    def __call__(self, *args, **kwargs):
        return self.f(self.obj, *args, **kwargs)

    def __str__(self):
        return '<bound method %s.%s of %s>' % (self.own.__name__, self.f.__name__, self.obj)


class instance_method():
    def __init__(self, function):
        self.function = function

    def __get__(self, instance, owner):
        if instance:
            return BoundMethod(self.function, instance, owner)
        else:
            return UnboundMethod(self.function, owner)
```

#### staticmethod classmethod

已经理解了上面的bound和unbound的描述器实现，那么staticmethod和classmethod就很好理解了

```python
class my_classmethod(object):
    def __get__(self, obj, type=None):
        return partial(self.f, type)

    def __init__(self, f):
        self.f = f


class my_staticmethod(object):
    def __init__(self, f):
        self.f = f

    def __get__(self, instance, owner):
        return self.f
```

