* mock一个方法，记录一个方法被调用的次数参数等
* mock一个传入参数，记录参数被调用情况



##### 假装自己是一个类，包含方法与他相同

```python
class A(object):
  def fun1():
    pass

m = mock(spec=A)
m.fun1() # success
m.fun2() # error Mock object has no attribute 'fun2'
```

