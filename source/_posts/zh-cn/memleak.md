---
title: 记一次排查内存泄漏问题
date: 2025-05-14 19:57:24
updated:
keywords:
slug: memleak
cover: /image/memleak.png
top_image:
comments: false
maincolor:
categories:
  - 后端开发
tags:
  - 工作研究
typora-root-url: ./memleak
description: 工作上遇到的内存泄漏问题，记录从排查到修复的过程
---

  
  
最近新上线的业务服务，跑了一周发现，接口越来越慢，经过一顿分析排查下来，发现是有内存泄漏了...

# 发现问题

接口平均耗时越来越慢，起初以为是某个下游服务请求耗时变高了
但是根据监控看各环节处理的耗时也没什么大的变化。
然后开始从服务的 log 分析看有没有可疑的地方，甚至在各个可疑的环节都加了耗时监控。
最后确定业务处理环节没有任何问题。

当我开始打开 k8s pod 监控才发现，服务进程的内存已经涨到限制的临界值了。
把时间拉长到一周来看，进程内存在每天业务繁忙期间都会出现净增长，且比较缓慢。

![](pod-log.png)

# 定位内存泄漏点

已经知道存在内存泄漏了，那么如何定位发生泄漏的位置呢？
golang 为我们提供了进程的资源监控工具：**pprof**
通过 pprof 我们可以分析进程的内存分布情况，只需要提前在服务中添加收集指标的代码即可

例如：
我在 http 服务中添加一个 pprof 的接口

```go
import (
  "github.com/gin-contrib/pprof"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

    // 初始化你的服务并获取本机IP
	r := config.InitAll()
	ip := lanip.GetByPrefix()
	if ip == "" {
		log.Panic().Msg("get no ip to serve")
	}

    // 注册业务路由
	route.SetupMiddleware(r)
	route.Swagger(r)
	route.API(r)

    // 注册pprof 路由
	pprof.Register(r, "debug/pprof")

	addr := fmt.Sprintf("%s:%d", ip, config.Config.Port)
	srv := &http.Server{
		Addr:    addr,
		Handler: r.Handler(),
	}

  if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatal().Err(err).Msg("listern failed")
	}
}
```

通过往 gin 注册 debug/pprof 路由，实现收集程序资源信息的能力。

我们使用以下指令来下载 pprof 信息：
`curl http://<ip>:<port>/debug/pprof/heap > heap.out`
使用 web 方式打开：
`go tool pprof --http="127.0.0.1:8080" heap.out`

# 修复过程 1

从 web 展示的 heap 内存分配树看，在 go-metrics 包下的 newStandardMeter 函数分配了很多内存，占当前链路的 72%
从百分比和矩形框的大小看，这样的内存占比是整个 heap 树里最显眼的，且也是不合理的。

![](heap.png)

让我们看看这个函数都做了什么：

```go
func newStandardMeter() *StandardMeter {
	return &StandardMeter{
		snapshot:  &MeterSnapshot{},
		a1:        NewEWMA1(),
		a5:        NewEWMA5(),
		a15:       NewEWMA15(),
		startTime: time.Now(),
	}
}
```

以及它的上层调用：

```go
// NewMeter constructs a new StandardMeter and launches a goroutine.
// Be sure to call Stop() once the meter is of no use to allow for garbage collection.
func NewMeter() Meter {
	if UseNilMetrics {
		return NilMeter{}
	}
	m := newStandardMeter()
	arbiter.Lock()
	defer arbiter.Unlock()
	arbiter.meters[m] = struct{}{}
	if !arbiter.started {
		arbiter.started = true
		go arbiter.tick()
	}
	return m
}
```

看上去，这个函数是 go-metrics 包提供的一个 Meter 对象的新建函数，在调用这个函数时，会创建一个 Meter 对象，并且启动一个 goroutine 来定期更新 Meter 对象的状态。
而这个 Meter 对象被保存在一个全局的 arbiter.meters map 里。

> meter 是用于监控系统中计算某事件在单位时间内发生的速率的，比如每分钟的访问速率。
> 通常是在服务端创建 meter 进行收集，计算速率，最后上传到监控系统中。
> 因此一个 meter 对象的生命周期通常只有一个单位时间左右，当然也可以复用

newStandardMeter() 持续创建新的 Meter 对象，这本是一个短暂的监控指标，一般埋点的对象都只是短暂收集几秒到几分钟的指标，就上传到日志系统了。

不再使用的 Meter 对象应该会被 gc 回收才对，但从 pprof 的内存分配看，并没有被回收。
也就是说，不再使用的 Meter 对象，并没有调用 Stop() 方法，从 arbiter.meters map 中删除，导致 gc 认为它是一个活跃的对象，没有被回收。

```go
func (m *StandardMeter) Stop() {
	if atomic.CompareAndSwapUint32(&m.stopped, 0, 1) {
		arbiter.Lock()
		delete(arbiter.meters, m)
		arbiter.Unlock()
	}
}
```

因此我在业务代码中，找到旧 Meter 对象不再使用的位置，执行 Meter.Stop() 进行主动删除，释放引用给gc回收。
但当我更新代码再观察几天后发现，还是存在内存泄漏，不过比之前泄漏的更缓慢了。
看来还有其他问题...

# 修复过程 2

我开始怀疑还是 go-metrics 包的问题，但无论如何测试，都没发现问题。
直到我在 go-metrics 的上游测试发现，go-cache 存在一些极端情况，会导致自动回收过期的 key 的 OnEvicted() 函数不生效！
而我添加的 Meter.Stop() 正好就在自动回收函数 OnEvicted() 里调用的。

我把相关的 github issue 贴出来：[没有调用过期回收函数](https://github.com/patrickmn/go-cache/issues/48)

> 这是一个 go 的 localcache 包，作者已经不再维护了

整个 bug 触发的过程是这样的：

1、新建一个 Meter 对象，我们把它保存在 go-cache 中，并设置过期时间，期望它在过期后能自动触发回收函数，并调用 Meter.Stop() 释放在 map 当中的引用。

2、当这个 Meter 对象过期了，但 go-cache 的回收协程此时还没扫描到这个 key，这时候 go-cache 触发新的 Set()动作，把这个 key 重新设置进去，覆盖掉了之前的 Meter，这就导致回收协程扫描到这个 key 的时候，这个 key 已经是重新活跃状态了，并且值是新的 Meter，旧的 Meter 就永远不会触发回收函数。

3、我们的 Meter 对象依靠回收函数来触发 Meter.Stop() 的目的达不到，就导致了 Meter 对象无法被回收。

这个情况发生的概率很低，也就解释了为什么内存虽然在泄漏，但是过程很慢，要一个星期左右才看出端倪。

修复过程也很简单，就是在 go-cache Set() 的时候，判断这个 key 是否存在并过期，是的话就执行删除触发 OnEvicted 回调，这样就能保证 Meter 对象被及时回收了。

```go
func (c *cache) Set(k string, x interface{}, d time.Duration) {
	// "Inlining" of set
	var e int64
	if d == DefaultExpiration {
		d = c.defaultExpiration
	}
	if d > 0 {
		e = time.Now().Add(d).UnixNano()
	}

	var item Item
	var evicted bool

	c.mu.Lock()
	if c.onEvicted != nil {
		item, evicted = c.items[k]
	}

	c.items[k] = Item{
		Object:     x,
		Expiration: e,
	}
	c.mu.Unlock()

	// 在这里判断旧的 item 是否过期，是的话就执行删除触发 OnEvicted 回调
	if evicted && item.Object != x {
		c.onEvicted(k, item.Object)
	}
}
```

# 总结

这次解决问题的过程，从看似一个很平常的接口耗时异常问题，牵引出一连串的排查思路和方向。

最后是因为没有主动调用 Stop() 以及 go-cache 没有触发 onEvicted() 回收函数导致的。

这次排查问题告诉我们一个道理，不要放过任何一个蛛丝马迹，利用好现有的工具，去仔细推敲，总能找到一个合理解释异常现象的理由。

事出必有因
