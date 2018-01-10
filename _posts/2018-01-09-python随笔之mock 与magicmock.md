---
layout: post
title: python随笔 Mock vs MagicMock 
description: mock 和magicmock的区别，以及何时使用。以及瞎扯淡
date: 2018.01.08 21:00 +08:00
tags: 
- python
- mock
---

首先，从现象上看，Mock有的MagicMock都有，而MagicMock多了一些magic方法。

```python
_mock = mock.Mock()
_magicmock = mock.MagicMock()
mock.__iter__ # error
_magicmock.__iter__ #<MagicMock name='mock.__iter__' id='4458398224'>
```

而且MagicMock是懒加载的，在没调用magic方法的时候，是不存在这个方法的。

```python
>>> _magicmock = mock.MagicMock()
>>> dir(_magicmock)
['assert_any_call', 'assert_called', 'assert_called_once'...
>>> iter(_magicmock)
<listiterator object at 0x10e32e290>
>>> dir(_magicmock)
['__iter__', 'assert_any_call', 'assert_called'...
```

很神奇。

所以在使用上，除了那种需要调用魔术方法报错的（如不允许迭代）都可以使用MagicMock。如果保守点，可以都使用Mock，其实也都够用来着。

---

下边进入正题，在看MagicMock中看到的有意思的东西。下边会回答MagicMock对魔术方法的懒加载是如何实现的。

首先，magicmock是mock的一个子类。

```python
class MagicMock(MagicMixin, Mock)
```

很明显，实现设置magic方法的在MagicMixin中。

```python
class MagicMixin(object):
    def __init__(self, *args, **kw):
        self._mock_set_magics()  # make magic work for kwargs in init
        _safe_super(MagicMixin, self).__init__(*args, **kw)
        self._mock_set_magics()  # fix magic broken by upper level init


    def _mock_set_magics(self):
        these_magics = _magics

        if getattr(self, "_mock_methods", None) is not None:
            these_magics = _magics.intersection(self._mock_methods)

            remove_magics = set()
            remove_magics = _magics - these_magics

            for entry in remove_magics:
                if entry in type(self).__dict__:
                    # remove unneeded magic methods
                    delattr(self, entry)

        # don't overwrite existing attributes if called a second time
        these_magics = these_magics - set(type(self).__dict__)

        _type = type(self)
        for entry in these_magics:
            setattr(_type, entry, MagicProxy(entry, self))
```

这段代码也不长，直接贴上来了。

可以看到，通过一串东西，找到了所有需要添加的魔术方法，然后给这个类赋予了一个MagicProxy。这个proxy会在第一次调用的时候创建魔术方法。

这边有个问题，直接获取type赋值，为啥不会影响其他的mock嘞？很简单，每个magicmock的type都不一样。

```python
>>> type(mock.MagicMock()) == type(mock.MagicMock())
False
```

直觉告诉我，是什么地方有修改`__new__`的地方。果然，在很上层的父类，NonCallableMock中找到了这么个东西。

```python
    def __new__(cls, *args, **kw):
        # every instance has its own class
        # so we can create magic methods on the
        # class without stomping on other mocks
        new = type(cls.__name__, (cls,), {'__doc__': cls.__doc__})
        instance = object.__new__(new)
        return instance
```

看注释也很清楚了，就是为了magic方法才做的这个事，在创建的时候做了些手脚。

##### 结论

把上面的过程顺起来说一遍。

* 在创建的时候，修改了`__new__`方法，每个实例都有自己独一无二的实例。
* 在MagicMock初始化的过程中，在类中增加各magic方法的代理。
* 实例调用magic方法的时候，会在第一次调用proxy的时候创建魔术方法。