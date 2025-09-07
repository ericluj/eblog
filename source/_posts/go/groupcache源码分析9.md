---
title: groupcache源码分析9:groupcache.go
date: 2020-03-29 21:34:37
tags: groupcache
categories: 
- go
- groupcache
---

终于到了最后一个文件groupcache.go，跟项目同名，看着就知道它的重要性了。前面我们分析了那么多，这一篇就来看看如何利用那些零件，来具体去实现整个缓存逻辑。

<!-- more -->

1. Getter接口，又一个Get方法，根据key查询到对应值，保存到dest中。GetterFunc是一个实现了Getter接口的func类型。
``` go
type Getter interface {
	Get(ctx context.Context, key string, dest Sink) error
}

type GetterFunc func(ctx context.Context, key string, dest Sink) error

func (f GetterFunc) Get(ctx context.Context, key string, dest Sink) error {
	return f(ctx, key, dest)
}
```

2. 定义一些使用到的变量。groups保存group与其对应结构体，initPeerServerOnce是一个sync.Once，它能保证Do方法只会被执行一次，实际上就是保证initPeerServer只会被执行一次。
``` go
var (
	mu     sync.RWMutex
	groups = make(map[string]*Group)

	initPeerServerOnce sync.Once
	initPeerServer     func()
)
```

3. 读锁并获取group名称对应的对象。
``` go
func GetGroup(name string) *Group {
	mu.RLock()
	g := groups[name]
	mu.RUnlock()
	return g
}
```

4. 创建Group，名称需保证唯一。
``` go
func NewGroup(name string, cacheBytes int64, getter Getter) *Group {
	return newGroup(name, cacheBytes, getter, nil)
}

func newGroup(name string, cacheBytes int64, getter Getter, peers PeerPicker) *Group {
	if getter == nil {
		panic("nil Getter")
	}
	mu.Lock()
	defer mu.Unlock()
	initPeerServerOnce.Do(callInitPeerServer) //保证callInitPeerServer只会被调用一次
	if _, dup := groups[name]; dup {
		panic("duplicate registration of group " + name)
	}
	g := &Group{
		name:       name,
		getter:     getter,
		peers:      peers,
		cacheBytes: cacheBytes,
		loadGroup:  &singleflight.Group{},
	}
	if fn := newGroupHook; fn != nil { //钩子方法
		fn(g)
	}
	groups[name] = g
	return g
}
```

5. 创建Group时用到的几个关联项。
``` go
var newGroupHook func(*Group) //钩子，创建Group时被调用。

func RegisterNewGroupHook(fn func(*Group)) { 
	if newGroupHook != nil {
		panic("RegisterNewGroupHook called more than once")
	}
	newGroupHook = fn
}

func RegisterServerStart(fn func()) {
	if initPeerServer != nil {
		panic("RegisterServerStart called more than once")
	}
	initPeerServer = fn
}

func callInitPeerServer() { //钩子，当第一个Group被创建时调用。
	if initPeerServer != nil {
		initPeerServer()
	}
}
```

6. Group结构体的定义。
``` go
type Group struct {
	name       string //名称
	getter     Getter //获取缓存的方法
	peersOnce  sync.Once
	peers      PeerPicker
	cacheBytes int64 //缓存大小限制

	mainCache cache //属于当前peer的缓存
	hotCache cache //属于其他peer的缓存，但是被查询当前peer额外保存一份

	loadGroup flightGroup //竞争请求，前面的singleflight.go

	_ int32 
	Stats Stats //统计值
}

type flightGroup interface {
	Do(key string, fn func() (interface{}, error)) (interface{}, error)
}

type Stats struct {
	Gets           AtomicInt //get请求总次数
	CacheHits      AtomicInt //从mainCache或hotCache命中的次数
	PeerLoads      AtomicInt //从其他peer获取数据的次数
	PeerErrors     AtomicInt //从其他peer获取数据错误的次数
	Loads          AtomicInt //非命中本peer的cache次数
	LoadsDeduped   AtomicInt //同一时间多请求只记一次
	LocalLoads     AtomicInt //从local获取数据总次数
	LocalLoadErrs  AtomicInt //从local获取数据错误次数
	ServerRequests AtomicInt //peer的所有http请求总次数
}
```

7. Name方法返回名称。initPeers对peers属性赋值。
``` go
func (g *Group) Name() string {
	return g.name
}

func (g *Group) initPeers() {
	if g.peers == nil {
		g.peers = getPeers(g.name)
	}
}
```

8. 这个方法是Group，根据参数key查询数据，然后将值放到dest里面。这里要注意下destPopulated的逻辑。
``` go
func (g *Group) Get(ctx context.Context, key string, dest Sink) error {
	g.peersOnce.Do(g.initPeers) //保证initPeers只被执行一次
	g.Stats.Gets.Add(1) //统计http总数量
	if dest == nil {
		return errors.New("groupcache: nil dest Sink")
	}
	value, cacheHit := g.lookupCache(key) //从mainCache和hotCache中查询

	if cacheHit { //查询到统计+1并返回数据
		g.Stats.CacheHits.Add(1)
		return setSinkView(dest, value)
	}

	destPopulated := false
    //同时多个请求，只有真正执行了的那个call，才会destPopulated返回true
    //为避免对dest中的值（实际时指针）重复赋值，只需要执行一次
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

9. 依次从mainCache和hotCache获取数据。
``` go
func (g *Group) lookupCache(key string) (value ByteView, ok bool) {
	if g.cacheBytes <= 0 {
		return
	}
	value, ok = g.mainCache.get(key)
	if ok {
		return
	}
	value, ok = g.hotCache.get(key)
	return
}
```

10. 加载数据。

Do方法中又再次进行了lookupCache，注释里是这么说的，singleflight只能对同时重叠的调用进行处理，假设有两个请求同时错过了cache，会导致load被调用两次，不幸的情况会导致cache.nbytes做出错误的计算。

我们梳理一下上面这段话，按照singleflight的逻辑，如果两个请求同时进入了Do方法，因为lock的缘故，第一个获的锁的执行，第二个等待锁释放，然后拿到call的返回值，实际并未执行。一开始我没想通，这样冲突不是不存在吗，为啥还要lookupCache一次呢？事实上，可能存在这一种情况，两个请求过来都没查到缓存，然后同时进入load方法，假如现在第一个执行的比较快，在第二个还没有获取锁就执行完毕退出了，则请求二成功获取锁，执行操作并且增加cache.nbytes，那么就会计算不正确了。
``` go
func (g *Group) load(ctx context.Context, key string, dest Sink) (value ByteView, destPopulated bool, err error) {
	g.Stats.Loads.Add(1)
	viewi, err := g.loadGroup.Do(key, func() (interface{}, error) {
		if value, cacheHit := g.lookupCache(key); cacheHit {
			g.Stats.CacheHits.Add(1)
			return value, nil
		}
		g.Stats.LoadsDeduped.Add(1)
		var value ByteView
		var err error
		if peer, ok := g.peers.PickPeer(key); ok { //获取peer，如果peer是自身返回nil
			value, err = g.getFromPeer(ctx, peer, key) //从peer获取值
			if err == nil {
				g.Stats.PeerLoads.Add(1)
				return value, nil
			}
			g.Stats.PeerErrors.Add(1)			
		}
		value, err = g.getLocally(ctx, key, dest) //从本地获取数据
		if err != nil {
			g.Stats.LocalLoadErrs.Add(1)
			return nil, err
		}
		g.Stats.LocalLoads.Add(1)
		destPopulated = true // dest已经被填充
		g.populateCache(key, value, &g.mainCache) //数据加到mainCache中
		return value, nil
	})
	if err == nil {
		value = viewi.(ByteView)
	}
	return
}
```

11. 从其他peer获取数据，peer.Get实际就是httpGetter的Get方法。这里使用了一个随机函数，一定概率会将其放入hotCache。
``` go
func (g *Group) getFromPeer(ctx context.Context, peer ProtoGetter, key string) (ByteView, error) {
	req := &pb.GetRequest{
		Group: &g.name,
		Key:   &key,
	}
	res := &pb.GetResponse{}
	err := peer.Get(ctx, req, res)
	if err != nil {
		return ByteView{}, err
	}
	value := ByteView{b: res.Value}
	if rand.Intn(10) == 0 {
		g.populateCache(key, value, &g.hotCache)
	}
	return value, nil
}
```

12. getLocally中实际调用的Get方法是我们在创建Group的时候去设定的，我们会在后面实际使用中介绍。
``` go
func (g *Group) getLocally(ctx context.Context, key string, dest Sink) (ByteView, error) {
	err := g.getter.Get(ctx, key, dest)
	if err != nil {
		return ByteView{}, err
	}
	return dest.view()
}
```

13. 设置缓存。假如当前缓存总大小超过了上线，那么使用lru来去除最老的值。
``` go
func (g *Group) populateCache(key string, value ByteView, cache *cache) {
	if g.cacheBytes <= 0 {
		return
	}
	cache.add(key, value)

	for {
		mainBytes := g.mainCache.bytes()
		hotBytes := g.hotCache.bytes()
		if mainBytes+hotBytes <= g.cacheBytes {
			return
		}

		victim := &g.mainCache
		if hotBytes > mainBytes/8 {
			victim = &g.hotCache
		}
		victim.removeOldest()
	}
}
```

14. 常量定义。
``` go
type CacheType int

const (
	MainCache CacheType = iota + 1
	HotCache
)
```

15. 返回Group中的缓存统计信息。
``` go
func (g *Group) CacheStats(which CacheType) CacheStats {
	switch which {
	case MainCache:
		return g.mainCache.stats()
	case HotCache:
		return g.hotCache.stats()
	default:
		return CacheStats{}
	}
}
```

16. cache结构体定义，与统计信息返回方法。
``` go
type cache struct {
	mu         sync.RWMutex
	nbytes     int64 // 缓存大小
	lru        *lru.Cache //缓存主体，lru
	nhit, nget int64 //命中和请求数
	nevict     int64 // 驱逐数
}

func (c *cache) stats() CacheStats {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return CacheStats{
		Bytes:     c.nbytes,
		Items:     c.itemsLocked(),
		Gets:      c.nget,
		Hits:      c.nhit,
		Evictions: c.nevict,
	}
}

type CacheStats struct {
	Bytes     int64
	Items     int64
	Gets      int64
	Hits      int64
	Evictions int64
}
```

17. 添加缓存方法，基于lru的Add。注意这里的nbytes计算，包含key和val的总长度。
``` go
func (c *cache) add(key string, value ByteView) {
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.lru == nil {
		c.lru = &lru.Cache{
			OnEvicted: func(key lru.Key, value interface{}) {
				val := value.(ByteView)
				c.nbytes -= int64(len(key.(string))) + int64(val.Len())
				c.nevict++
			},
		}
	}
	c.lru.Add(key, value)
	c.nbytes += int64(len(key)) + int64(value.Len())
}
```

18. 获取缓存。
``` go
func (c *cache) get(key string) (value ByteView, ok bool) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.nget++
	if c.lru == nil {
		return
	}
	vi, ok := c.lru.Get(key)
	if !ok {
		return
	}
	c.nhit++
	return vi.(ByteView), true
}
```

19. 删除老旧数据。
``` go
func (c *cache) removeOldest() {
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.lru != nil {
		c.lru.RemoveOldest()
	}
}
```

20. 获取缓存总大小和总数量。
``` go
func (c *cache) bytes() int64 {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return c.nbytes
}

func (c *cache) items() int64 {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return c.itemsLocked()
}

func (c *cache) itemsLocked() int64 {
	if c.lru == nil {
		return 0
	}
	return int64(c.lru.Len())
}
```

21. 封装方法，用来完成对int64的原子操作。
``` go
type AtomicInt int64

func (i *AtomicInt) Add(n int64) {
	atomic.AddInt64((*int64)(i), n)
}

func (i *AtomicInt) Get() int64 {
	return atomic.LoadInt64((*int64)(i))
}

func (i *AtomicInt) String() string {
	return strconv.FormatInt(i.Get(), 10)
}
```