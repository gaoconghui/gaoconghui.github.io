---
layout: post
title: groupcache源码中几个有趣的点
description: 读groupcache源码过程中的一点笔记
date: 2018.06.06 21:00 +08:00
tags: 
- go
---



### 简介

>groupcache is a caching and cache-filling library, intended as a replacement for memcached in many cases.

groupcache是一个可分布式缓存组件，用于在某些方面替代memcache，不过和一般的缓存有些区别，它只能做get操作（没错，只能get），但是不能做更新和删除操作。另外，groupcache可以很方便的集成到应用程序中，用http接口的形式与其他程序交互。

#### 几个概念

* peer 代表一个缓存服务器
* group 缓存以group的概念做组的划分，可以有多个group，每个group有不同的获取数据的方法

#### 几个文件

* sinks.go 用来装cache的值，可以支持多种类型。


* consistenthash/consistenthash.go 一个一致性hash的实现


* http.go 提供peer之间的交互。依赖`consistenthash`提供的一致性hash的方案，查找key对应的缓存服务器并发出请求，以及提供接口给其他peer访问
* peers.go 提供了几个和peer相关的方法，如根据groupname获取peer
* groupcache.go 核心文件，提供获取缓存的方法（先查找本地再查找远端）
* singleflight/singleflight.go 提供一个竞争执行方法的方法
* lru/lru.go 顾名思义 ，提供lru的实现

### 基本使用方式

**创建group**

```go
// 需要提供一个获取的getter方法
var Group = groupcache.NewGroup("colorCache", 64<<20, groupcache.GetterFunc(
	func(ctx groupcache.Context, key string, dest groupcache.Sink) error {
		log.Println("looking up", key)
      	// 这里从数据库中获取一个值 
		v, ok := Store[key]
		if !ok {
			return errors.New("color not found")
		}
		dest.SetBytes(v)
		return nil
	},
))
```

**设置peers并启动服务**

```go
// peer是本缓存服务服务的地址 如http://localhost:8080
pool := groupcache.NewHTTPPool(peer)
// ps是一组缓存服务器的地址
pool.Set(ps...)
http.ListenAndServe(*addr, nil)
```

**获取缓存**

```go
var b []byte
// 获取缓存结果，存在b中。AllocatingByteSliceSink是一个对获取数据的包装
err := Group.Get(nil, color, groupcache.AllocatingByteSliceSink(&b))
```

### 几个有趣的点

#### peer的查询

给定一个key，groupcache会在本地找不到缓存的情况下，查询该key应该存在的peer。为了在新增或删除peer的时候尽量少的缓存失效，groupcache使用一致性hash的方案，并提供了一个consistenthash的实现，就在consistenthash/consistenthash.go中。

不赘述什么是一致性hash了，直接以注释分析的方式给出代码

```go
type Hash func(data []byte) uint32

// hash 一个hash算法，默认为crc32.ChecksumIEEE
// replicas 每个peer节点会产生几个虚拟节点
// keys 储存所有虚拟节点的hash值
// hashMap 根据hash值找到相应的peer
type Map struct {
	hash     Hash
	replicas int
	keys     []int // Sorted
	hashMap  map[int]string
}

func New(replicas int, fn Hash) *Map {
	m := &Map{
		replicas: replicas,
		hash:     fn,
		hashMap:  make(map[int]string),
	}
	if m.hash == nil {
		m.hash = crc32.ChecksumIEEE
	}
	return m
}

// Returns true if there are no items available.
func (m *Map) IsEmpty() bool {
	return len(m.keys) == 0
}

// Adds some keys to the hash.
func (m *Map) Add(keys ...string) {
	for _, key := range keys {
      // 一个key就是一个peer节点
      // 对于每个key，都会生成replicas个虚拟节点，并把虚拟节点和key的对应关系存起来
		for i := 0; i < m.replicas; i++ {
			hash := int(m.hash([]byte(strconv.Itoa(i) + key)))
			m.keys = append(m.keys, hash)
			m.hashMap[hash] = key
		}
	}
  // 排序 方便后边查找
	sort.Ints(m.keys)
}

// 这边的key为一个cache中的key，即要查询的key
// 计算这个key的hash值，找到与之最为接近的虚拟节点，再通过虚拟节点找到具体的peer
func (m *Map) Get(key string) string {
	if m.IsEmpty() {
		return ""
	}

  // 计算key对应的hash值
	hash := int(m.hash([]byte(key)))

	// 使用二分法查找key对应的hash值最近的虚拟节点
	idx := sort.Search(len(m.keys), func(i int) bool { return m.keys[i] >= hash })

	// 一致性hash的虚拟节点应该构成一个环
	if idx == len(m.keys) {
		idx = 0
	}

	return m.hashMap[m.keys[idx]]
}
```

很经典的实现。

#### cache的查找过程

简单的说，就是`Group.Get(nil, color, groupcache.AllocatingByteSliceSink(&b))`发生了什么。

```go
func (g *Group) Get(ctx Context, key string, dest Sink) error {
  // ... (省略一些不重要的代码)
  // 查询本地的缓存，很简单的方法
	value, cacheHit := g.lookupCache(key)

  // 如果命中，统计完直接返回
	if cacheHit {
		g.Stats.CacheHits.Add(1)
		return setSinkView(dest, value)
	}

	// 这边是关键 
	destPopulated := false
	value, destPopulated, err := g.load(ctx, key, dest)
	if err != nil {
		return err
	}
	if destPopulated {
		return nil
	}
	return setSinkView(dest, value)
}

```

如代码的注释，关键在于在本地没找到缓存的情况下，需要找到合适的peer（可能是自己）去计算出值。这里需要考虑一个问题，就是如果本地有两个协程同时查询一个key怎么办，groupcache会阻塞其中一个，让另一个先跑出结果，避免调用两次获取方法。`destPopulated`标记该值是不是由该peer新计算出来的（本地方法获取后会直接赋值dest）。

让我们看一下这边的load方法

```go
func (g *Group) load(ctx Context, key string, dest Sink) (value ByteView, destPopulated bool, err error) {
	g.Stats.Loads.Add(1)
  // loadGroup.Do是个竞争的方法，相同的key同时只会有一个访问
	viewi, err := g.loadGroup.Do(key, func() (interface{}, error) {
      // 可能存在两个并发访问了这个方法，后执行的协程可能可以直接从缓存中读入
		if value, cacheHit := g.lookupCache(key); cacheHit {
			g.Stats.CacheHits.Add(1)
			return value, nil
		}
		g.Stats.LoadsDeduped.Add(1)
		var value ByteView
		var err error
      // 根据上边的一致性hash，找到peer
		if peer, ok := g.peers.PickPeer(key); ok {
          // 从peer中获取值
			value, err = g.getFromPeer(ctx, peer, key)
			if err == nil {
				g.Stats.PeerLoads.Add(1)
				return value, nil
			}
			g.Stats.PeerErrors.Add(1)
		}
      // 不需要访问远端peer，可以直接自己获取
		value, err = g.getLocally(ctx, key, dest)
		if err != nil {
			g.Stats.LocalLoadErrs.Add(1)
			return nil, err
		}
		g.Stats.LocalLoads.Add(1)
		destPopulated = true // only one caller of load gets this return value
		// 缓存起来
      	 g.populateCache(key, value, &g.mainCache)
		return value, nil
	})
	if err == nil {
		value = viewi.(ByteView)
	}
	return
}

```

还需要解释最后一个问题，如何控制并发的访问。这边使用的是`g.loadGroup.Do`方法，`loadGroup`是一个`singleflight.Group`。让我们看一下这个Do方法做了啥。

```go
package singleflight

import "sync"

// 包装一个key获取值锁需要的一些参数
type call struct {
	wg  sync.WaitGroup
	val interface{}
	err error
}

type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}

func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
  // 懒加载
	if g.m == nil {
		g.m = make(map[string]*call)
	}
  // 已经有协程在处理了，阻塞(c.wg.Wait())等待完成
	if c, ok := g.m[key]; ok {
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err
	}
  // 目前没有协程在处理，新建一个处理的任务
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	g.mu.Unlock()

	return c.val, c.err
}
```

这个方法还是比较简单的，主要就是使用`sync.WaitGroup`来保证只有一个协程在真的调用`fn`方法获取值，剩下的都在等待获取完成。

### 小结

看groupcache源码纯粹只是为了学习优秀的go的代码，并没有在实际中使用过… 不过这代码还是很有意思的。