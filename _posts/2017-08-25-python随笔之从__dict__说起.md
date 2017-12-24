---
layout: post
title: python随笔之从__dict__说起
date: 2017.08.25 21:00 +08:00
tags: 
- python随笔
---

先上一段代码，来源是[github](https://github.com/faif/python-patterns/blob/master/creational/borg.py)。

```python
class Borg(object):
    __shared_state = {}

    def __init__(self):
        self.__dict__ = self.__shared_state
        self.state = 'Init'

    def __str__(self):
        return self.state

class YourBorg(Borg):
    pass

if __name__ == '__main__':
    rm1 = Borg()
    rm2 = Borg()

    rm1.state = 'Idle'
    rm2.state = 'Running'

    print('rm1: {0}'.format(rm1))
    print('rm2: {0}'.format(rm2))

    rm2.state = 'Zombie'

    print('rm1: {0}'.format(rm1))
    print('rm2: {0}'.format(rm2))

    print('rm1 id: {0}'.format(id(rm1)))
    print('rm2 id: {0}'.format(id(rm2)))

    rm3 = YourBorg()

    print('rm1: {0}'.format(rm1))
    print('rm2: {0}'.format(rm2))
    print('rm3: {0}'.format(rm3))

### OUTPUT ###
# rm1: Running
# rm2: Running
# rm1: Zombie
# rm2: Zombie
# rm1 id: 140732837899224
# rm2 id: 140732837899296
# rm1: Init
# rm2: Init
# rm3: Init
```

上面这一段代码，乍看挺神奇的，Borg 的各个实例共享了state。实现起来也很巧妙，利用了\_\_dict\_\_。 我们知道，python中\_\_dict\_\_存储了该对象的一些属性。类和实例分别拥有自己的\_\_dict\_\_，且实例会共享类的\_\_dict\_\_。


这里有一个我一直以来都搞混的知识点，在\_\_init\_\_ 中声明的变量 ，以及在方法体之外声明的变量分别是在哪里。很简单的测试就能得到，在\_\_init\_\_中，self.xxx = xxx会把变量存在实例的\_\_dict\_\_中，仅会在该实例中能获取到，而在方法体外声明的，会在class的\_\_dict\_\_中。下面的简单的测试。

```
class MyclassA():

    in_class = {}

    def __init__(self):
        self.in_func = {}

a = MyclassA()


print MyclassA.__dict__  # {'in_class': {}, '__module__': '__main__', '__doc__': None, '__init__': <function __init__ at 0x10eb040c8>}
print a.__dict__ # {'in_func': {}}

```
---
有个有意思的问题，调用obj.key的搜索顺序是咋样的？
#### attr的搜索顺序
这个很有意思，比我想象的要复杂很多，因为涉及到了descriptor，不仅仅只是查询实例或者是类的\_\_dict\_\_这么简单，直接上个结论


```
1.如果attr是一个Python自动产生的属性，找到！(优先级非常高！)
2.查找obj.__class__.__dict__，如果attr存在并且是data descriptor，返回data descriptor的__get__方法的结果，如果没有继续在obj.__class__的父类以及祖先类中寻找data descriptor
3.在obj.__dict__中查找，这一步分两种情况，第一种情况是obj是一个普通实例，找到就直接返回，找不到进行下一步。第二种情况是obj是一个类，依次在obj和它的父类、祖先类的__dict__中查找，如果找到一个descriptor就返回descriptor的__get__方法的结果，否则直接返回attr。如果没有找到，进行下一步。
4.在obj.__class__.__dict__中查找，如果找到了一个descriptor(插一句：这里的descriptor一定是non-data descriptor，如果它是data descriptor，第二步就找到它了)descriptor的__get__方法的结果。如果找到一个普通属性，直接返回属性值。如果没找到，进行下一步。
5.很不幸，Python终于受不了。在这一步，它raise AttributeError
```
用代码解释上面的过程(这其中包含了descriptor相关概念)

```python
作者：刘缙
链接：https://www.zhihu.com/question/25391709/answer/30634637
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

def object_getattr(obj, name):
    'Look for attribute /name/ in object /obj/.'
    # First look in class and base classes.
    v, cls = class_lookup(obj.__class__, name)
    if (v is not None) and hasattr(v, '__get__') and hasattr(v, '__set__'):
        # Data descriptor.  Overrides instance member.
        return v.__get__(obj, cls)
    w = obj.__dict__.get(name)
    if w is not None:
        # Found in object
        return w
    if v is not None:
        if hasattr(v, '__get__'):
            # Function-like descriptor.
            return v.__get__(obj, cls)
        else:
            # Normal data member in class
            return v
    raise AttributeError(obj.__class__, name)
    
 def class_lookup(cls, name): 
     'Look for attribute /name/ in class /cls/ and bases.' 
     v = cls.__dict__.get(name) 
     if v is not None: 
         # found in this class 
         return v, cls 
     # search in base classes 
     for i in cls.__bases__: 
         v, c = class_lookup(i, name) 
         if v is not None: 
             return v, c 
     # not found 
     return None

```

经常看到有人说obj.key = val 等价于obj.\_\_dict\_\_[key] = val，其实根据我们上面的那个搜索顺序，在有descriptor的情况下，会不太一样。

```python
class DataDescriptor(object):
    def __init__(self, init_value):
        self.value = init_value

    def __get__(self, instance, owner):
        return "DataDescriptor __get__" + str(self.value)

    def __set__(self, instance, value):
        self.value = value
        print "DataDescriptor __set__"

class ClassA(object):
    val_descriptor = DataDescriptor(0)
    val_normal = 0

if __name__ == '__main__':
    a = ClassA()
    print a.val_descriptor # DataDescriptor __get__0
    print a.val_normal # 0
    a.val_descriptor = "has change"
    a.val_normal = "has change"
    print a.val_descriptor  # DataDescriptor __get__has change
    print a.val_normal # has change
    a.__dict__["val_descriptor"] = "change again"
    a.__dict__["val_normal"] = "change again"
    print a.val_descriptor # DataDescriptor __get__has change
    print a.val_normal #change again
```

重点是最后两行行，要修改的是一个descriptor 的话，obj.key = val 的方式是而已修改成功的，会调用descriptor 的\_\_set\_\_ 方法，而直接修改\_\_dict\_\_的方式则会有问题，因为在搜索attr的时候会先查找class的descriptor（对，即便`a.__dict__["val_descriptor"] = DataDescriptor("change again")`这样子也是没用的）。

---

关于\_\_dict\_\_，还有一个有趣的点。cls.\_\_dict\_\_ 是一个dictproxy，obj.\_\_dict\_\_ 是一个dict，前者是一个只读对象，后者是一般的dict。

这意味着，对于一个实例，可以使用obj.\_\_dict\_\_[key] = val的方式去赋值，也可以使用obj.key = value的方式赋值（这种方式会调用\_\_setattr\_\_方法，会按照attr的搜索顺序去搜索），而cls只能使用后者，即必须经过\_\_setattr\_\_ ，stackoverflow 上有人解释说是为了编译器优化。

个人认为这种设置还有另外一种原因，即保护了descriptor的使用。

看上面那长段的代码，我们发现，如果一个类中设置了一个descriptor，后续没有任何办法绕过descriptor的\_\_set\_\_方法。
* 使用obj.key = val 的方式，毫无以为会调用descriptor的\_\_set\_\_方法。
* obj.\_\_dict\_\_[key] = val的方式，会修改实例的\_\_dict\_\_，但是按照搜索的顺序会先搜索类的descriptor，所以相当于没有修改
* cls.key = val，会调用descriptor的\_\_set\_\_方法
* 于是只剩下直接修改cls.\_\_dict\_\_，但很可惜，这是一个只读对象无法修改。

### 后记
非常的乱，这些东西都是我在看python的设计模式的时候逐渐发现问题慢慢搜索的，主要是为了给自己做个记录，当做一个随笔，不太适合除了我以外的人看... 有空的时候会整理下。