---
layout: post
title: python实现redis三种cas操作
date: 2017.12.08 21:10 +08:00
---

cas全称是compare and set，是一种典型的事务操作，本文会介绍三种redis实现cas事务的方法，并会解决下面的虚拟问题：
维护一个值，如果这个值小于当前时间，则设置为当前时间；如果这个值大于当前时间，则设置为当前时间+30。简单的单线程环境下代码如下:

```
# 初始化
r = redis.Redis()
if not r.exists("key_test"):
    r.set("key_test", 0)

def inc():
    count = int(r.get('key_test')) + 30 #1
    # 如果值比当前时间小，则设置为当前时间
    count = max(count, int(time.time())) #2
    r.set('key_test', count) #3
    return count
```

很简单的一段代码，在单线程环境下可以跑的很欢，但显然，是无法移植到多线程或者是多进程环境的（进程A和B同时运行到#1，获取了相同的count值，然后运行#2#3，会导致count值总共只增加了30）。而为了能在多进程环境下运行，我们需要引入一些其他的东西。
### py-redis本身自带的事务操作
redis有这么几个和事务相关的命令，multi,exec,watch。通过这几个命令，可以实现‘将多个命令打包，然后一次性、按顺序执行，且不会被终端’。事务会从MULTI开始，执行EXEC后触发事件。另外，我们还需要WATCH，watch可以监视任意数量的键，当在调用EXEC执行事务时，如果任意一个键被修改了，整个事务不会执行。

下边是使用redis本身的事务解决cas问题的代码。

```
class CasNormal(object):
    def __init__(self, host, key):
        self.r = redis.Redis(host)
        self.key = key
        if not self.r.exists(self.key):
            self.r.set(self.key, 0)

    def inc(self):
        with self.r.pipeline() as pipe:
            while True:
                try:
            		#监视一个key，如果在执行期间被修改了，会抛出WatchError
                    pipe.watch(self.key)
                    next_count = 30 + int(pipe.get(self.key))
                    pipe.multi()
                    if next_count < int(time.time()):
                        next_count = int(time.time())
                    pipe.set(self.key, next_count)
                    pipe.execute()
                    return next_count
                except WatchError:
                    continue
                finally:
                    pipe.reset()
```

代码也不复杂，引入了之前说到的multi,exec,watch，如果对事务操作比较熟悉的同学，可以很容易看出来，这是一个乐观锁的操作（咱们假设没人竞争来着，每次去拿数据的时候都不会上锁，真有人来改了再说。）乐观锁在高并发的情况下会显得很无力，文末的性能对比会显示这个问题。

### 使用基于redis的悲观锁
悲观锁，就是很悲观的锁，每次拿数据都会假设别人也要拿，先给锁起来，用完再把锁释放掉。redis本身没有实现悲观锁，但我们可以先用redis实现一个悲观锁。

>此处应该有个推倒出redis悲观锁的过程，不过太麻烦了...直接丢个链接吧...
https://gist.github.com/gaoconghui/61e878c725952c134a1193d560df7434

ok，咱们现在有悲观锁了，做起事来也有底气了，根据上边的代码，咱们只要加上@ synchronized注释就能保证同一时间只有一个进程在执行。下边是基于悲观锁的解决方案。

```
lock_conn = redis.Redis("localhost")

class CasLock(object):
    def __init__(self, host, key):
        self.r = redis.Redis(host)
        self.key = key
        if not self.r.exists(self.key):
            self.r.set(self.key, 0)

    @synchronized(lock_conn, "lock", 10)
    def inc(self):
        next_count = 30 + int(self.r.get(self.key))
        if next_count < int(time.time()):
            next_count = int(time.time())
        self.r.set(self.key, next_count)
        return next_count
```
代码看上去少多了（因为引入了synchronized...）

### 基于lua脚本实现
上边两种方法都是用锁来实现的，锁的实现总会出现竞争的问题，区别无非是出现竞争了咋办的问题。使用redis lua脚本的实现，可以直接把这个cas操作当成一个<b>原子操作</b>。

我们知道，redis本身的一系列操作，都是原子操作，且redis会按顺序执行所有收到的命令。先看代码

```
class CasLua(object):
    def __init__(self, host, key):
        self.r = redis.Redis(host)
        self.key = key
        if not self.r.exists(self.key):
            self.r.set(self.key, 0)
        self._lua = self.r.register_script("""
        local next_count = redis.call('get',KEYS[1]) + ARGV[1]
        ARGV[2] = tonumber(ARGV[2])
        if next_count < ARGV[2] then
        	next_count = ARGV[2]
        end
        redis.call('set',KEYS[1],next_count)
        return tostring(next_count)
                """)

    def inc(self):
        return int(self._lua([self.key], [30, int(time.time())]))
```

这里先注册了这个脚本，后边可以直接去使用他。关于redis lua脚本的文章有不少，感兴趣的可以去搜搜看，这边就不赘述了。

### 性能对比
这边的测试只是一个非常简单的测试（不过还是能看出效果来的），测试换机就是自己的开发机，数字看个大小就行了。

分别测了三种操作在单线程，五个线程，十个线程，五十个线程情况下，进行1000次操作各自的表现，时间如下


| 线程数  | 乐观锁实现 | 悲观锁实现  | lua|
| :------: |:--------:| :--------:|:--------:|
| 1      | 0.43 			  | 0.71| 0.35|
| 5      | 5.80    		  |   3.10 | 0.62|
| 10 	  | 17.80	         |    5.60 | 1.30|
| 50 	  | 245.00        |    29.60 | 6.50|

依次是redis本身事务实现的乐观锁，基于redis实现的悲观锁以及lua实现。

在比较悲观锁和乐观锁之前，需要先说明一点，这边的测试对乐观锁不是很公平，乐观锁本身就是假设不会有很多的并发的。在单线程情况下，悲观锁要差一些。单线程下，不存在竞争关系，悲观锁耗时长仅因为是多了一次redis的网络交互。随着线程的增加，悲观锁的性能逐渐变好，毕竟悲观锁本身就是为了解决这种高并发高竞争的环境而诞生的。在50线程的时候，乐观锁的实现单次操作的时间要0.245秒，非常恐怖，如果是生产环境，几乎都不能用了。

至于lua的性能，快的不可思议，几乎就是线性增加。（50线程的情况下，平均的1000次完成时间是6.5s，换言之，6.5秒内执行了50 * 1000次cas操作）。

以上测试都是本地redis，本地测试，如果redis是远端的，网络交互时间会增加，lua优势会更加明显。

以上。