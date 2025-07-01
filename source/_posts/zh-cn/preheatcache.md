---
title: 缓存预热的实践
date: 2025-04-13 20:50:31
updated:
keywords:
slug: preheatcache
cover: /image/preheat.png
top_image:
comments: false
maincolor:
categories:
  - 后端开发
tags:
  - 工作研究
typora-root-url: ./preheatcache
---

当服务的流量达到一定量时，就需要引入缓存来保护底层的服务资源不被耗尽。

提起缓存，可以联想到客户端缓存，如浏览器的强缓存、协商缓存、session、localStorage
还有代理缓存，如 CDN 缓存，nginx 反向代理缓存
还有服务端缓存，如 redis、memcache、localcache 等缓存

今天要讲的是关于服务端的缓存预热的实践。

# 背景

由于业务的原因，每天的业务高峰期在早上 10 点开始持续到晚上 22 点，并且请求是在 10 点的时候突然开始大批量访问的。

服务每天都要处理 2k 并发量的请求，尽管 http 服务端已经做了 redis 缓存和本地缓存，同时加了缓存更新的双重检查锁，减少打到数据库的流量。

每天早上起量的时候，还是无法解决起量的几分钟内，很多请求因为双重检查锁的作用，从数据库获取基础数据超时的问题。

因此，需要引入缓存预热逻辑，在服务起量前加载数据到缓存中，顺利支撑起瞬时并发带来的问题。

# 双重检查

在服务查询缓存 -> 查询数据库中间，设置一个双重检查锁，实现请求查询不到缓存，等待一会，如果前面相同的查询从数据库取回数据，回填缓存，自己再从缓存中取的目的。
这样能减少很多不必要的查询数据库操作。

![](querycache.png)

如图，右边的流程加了锁，在锁的前后流程做了两次缓存检查，这样就能减少短时间内的重复查询请求进入数据库
这也是 “双重检查锁”的名字由来

在我的业务场景中，能容忍多一点请求进入到数据库，因此这里的锁我换成了令牌桶，能支持一定量的并发冲击到数据库，换取锁等待的耗时代价。

令牌桶的容量决定了并发冲击量的限制，因此根据你的服务资源能力来设置即可。

# 缓存预热

做了双重检查，还是会面临瞬时过大的请求量冲击导致查询耗时过高。
这时要面临保留双重检查锁保护数据库还是去掉锁，降低瞬时请求的耗时问题。

其实这里可以使用缓存预热，在业务中，瞬时起量的请求 key 可以通过每天收集服务的请求数据得知。
然后就可以用来做缓存预热了，这样就能使每天第一波流量冲击到来前，将数据载入缓存，减少了耗时。

下面是我在业务中实现的缓存 hot key 收集和预热的代码：

```go
package cache

import (
	"sort"
	"sync"
	"sync/atomic"
)

// 热点key 记录
type HotCacheKey struct {
	mu   sync.RWMutex
	keys map[string]*atomic.Uint32
}

type HotKeyVal struct {
	Key string
	Val uint32
}

func NewHotCacheKeyCaler() *HotCacheKey {
	return &HotCacheKey{
		mu:   sync.RWMutex{},
		keys: map[string]*atomic.Uint32{},
	}
}

func (k *HotCacheKey) SetkeyCountAsync(key string) {
	go func() {
		k.SetKeyCount(key)
	}()
}

func (k *HotCacheKey) SetKeyCount(key string) {
	k.mu.Lock()
	defer k.mu.Unlock()

	val, exist := k.keys[key]
	if !exist {
		newVal := new(atomic.Uint32)
		newVal.Store(1)
		k.keys[key] = newVal
		return
	}

	val.Add(1)
}

func (k *HotCacheKey) GetAndClearHotKeys(top int) []HotKeyVal {
	k.mu.RLock()

	var keyVals = []HotKeyVal{}
	for k, v := range k.keys {
		keyVals = append(keyVals, HotKeyVal{Key: k, Val: v.Load()})
	}
	k.keys = map[string]*atomic.Uint32{}
	k.mu.RUnlock()

	sort.Slice(keyVals, func(i, j int) bool {
		return keyVals[i].Val > keyVals[j].Val
	})

	if top > len(keyVals) {
		top = len(keyVals)
	}
	return keyVals[:top]
}

```

通过将 key 收集到 map 中进行计数，并且在服务设定的时间点，使用 cron 定时任务调用 GetAndClearHotKeys() 读取 hotkey ，来执行查询数据并回填到缓存中。

这里收集的 hotCacheKey 是放在服务进程中的，也可以引入 redis 将收集的 hotkey 存到 cache 中，这样每次重启服务就可以重新读取进来，继续收集

# 服务持续运行

在更新了缓存预热的逻辑后，服务在每天起量时，查询的耗时降了 80%。
恢复的正常的水平了，并且数据库也不会受到冲击。

缓存预热仅仅牺牲了一些进程存储空间，就做到很好的效果，非常值得。
