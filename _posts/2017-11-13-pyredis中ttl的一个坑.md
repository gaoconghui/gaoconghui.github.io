---
layout: post
title: pyredis中ttl的坑
description: pyredis中，ttl函数显示过期的时间与实际key消失的时间存在0.5秒的差值
tags: pyredis
date: 2017-11-13 20:07
---



### 问题复现

在用py-redis 的redis类做redis悲观锁的时候发现了一个问题，ttl方法显示过期（返回None）的时候，redis中key实际上没有消失，大概还能存活0.5秒，但是如果换成strictRedis就没问题了，很有意思。下面是一个简单的复现的代码。

```python
r = redis.Redis()
lock_name = "test_lock"
r.set(lock_name, "test")
r.expire(lock_name, 1)

t1 = 0
t2 = 0
while True:
    # 过期
    if not r.ttl(lock_name):
        # 记录下第一次ttl返回None的时间
        if t1 == 0: t1 = time.time()
        # 记录键消失的时间
        if not r.exists(lock_name):
            t2 = time.time()
            break
    time.sleep(.01)
print t2 - t1
```

expire方法设置过期时间，ttl方法获取过期时间（不存在则为None），按理说过期之后，键就会消失，即t2 - t1 应该为0，但上面的case返回大约是0.5s。

### 解决方法
* 使用pttl（返回毫秒数）代替ttl
* 使用StrictRedis 代替Redis

### 原因
首先要确认一点，这是redis-py的锅，人家redis在某个版本之后已经能在1毫秒以内干掉过期的键了。具体是啥锅，得好好理一理

##### redis ttl正常的返回值
* 当 key 不存在时，返回 -2 。
* 当 key 存在但没有设置剩余生存时间时，返回 -1 。
* 否则，以秒为单位，返回 key 的剩余生存时间。
* 注意 上边以秒为单位的生存时间，在我测试后发现是四舍五入后的，如果时间为0.4秒则会为0

##### StrictRedis 和Redis 区别
* Redis 是StrictRedis 的子类，覆写了几个方法，大多是参数位置变一变，据说是为了兼容之前的版本所做的。

* Redis 中跟TTL 相关的是这样一句

  > ```
  > # Overridden callbacks
  > RESPONSE_CALLBACKS = dict_merge(
  >     StrictRedis.RESPONSE_CALLBACKS,
  >     {
  >         'TTL': lambda r: r >= 0 and r or None,
  >         'PTTL': lambda r: r >= 0 and r or None,
  >     }
  > )
  > ```

  这是一个解析redis返回的方法，StrictRedis 会直接返回值，而Redis会做一次包装。

问题就出在Redis 对redis返回的值的包装上，`lambda r: r >= 0 and r or None` 我猜测他本意应该是当对象存在ttl时返回ttl，不存在key或者没有设置expire时返回None（即返回值为-2 或者为-1 或者返回值就是None的时候），然而因为四舍五入的存在，0.4会变为0，`lambda r: r >= 0 and r or None`返回值则为None。私以为这里变为`lambda r: r >= 0 and r is not None or None`更为合适。

综上。