---
title: groupcache源码分析8
date: 2020-03-24 21:34:34
tags: groupcache
categories: 
- go
- groupcache
---

http.go定义了groupcache接收到http请求时怎么去执行操作，算是一个主要流程文件。

<!-- more -->

1. 定义两个常量，作为服务的默认值，defaultBasePath是请求路径，defaultReplicas是前面一致性哈希提到的单台机器虚拟结点数。
``` go
const defaultBasePath = "/_groupcache/"

const defaultReplicas = 50
```

2. HTTPPool为文件的核心，它是一个实现了PeerPicker接口的peer池。Context是上下文返回方法，如果没有设置使用默认的。Transport返回http.RoundTripper，没有同样使用默认的，它相当于http请求的中间件。self默认地址。opts为相关配置。改动peers和httpGetters时，通过mu来加锁。peers是之前一致性哈希章节中定义的结构体类型，用来存取peer。httpGetters存储地址与httpGetter对应关系。
``` go
type HTTPPool struct {
	Context func(*http.Request) context.Context

	Transport func(context.Context) http.RoundTripper

	self string

	opts HTTPPoolOptions

	mu          sync.Mutex 
	peers       *consistenthash.Map
	httpGetters map[string]*httpGetter
}
```

3. HTTPPool用到的配置项。HashFn是哈希计算方法。
``` go
type HTTPPoolOptions struct {
	BasePath string
	Replicas int
	HashFn consistenthash.Hash
}
```

4. 创建方法，NewHTTPPoolOpts返回一个*HTTPPool，因为它实现了ServeHTTP，被注册为http处理方法。httpPoolMade变量保证HTTPPool只会被初始化一次。
``` go
func NewHTTPPool(self string) *HTTPPool {
	p := NewHTTPPoolOpts(self, nil)
	http.Handle(p.opts.BasePath, p)
	return p
}

var httpPoolMade bool
```

5. NewHTTPPoolOpts逻辑不复杂，主要是为结构体赋值，HTTPPool实现了PeerPicker接口，通过RegisterPeerPicker注册。
``` go
func NewHTTPPoolOpts(self string, o *HTTPPoolOptions) *HTTPPool {
	if httpPoolMade {
		panic("groupcache: NewHTTPPool must be called only once")
	}
	httpPoolMade = true

	p := &HTTPPool{
		self:        self,
		httpGetters: make(map[string]*httpGetter),
	}
	if o != nil {
		p.opts = *o
	}
	if p.opts.BasePath == "" {
		p.opts.BasePath = defaultBasePath
	}
	if p.opts.Replicas == 0 {
		p.opts.Replicas = defaultReplicas
	}
	p.peers = consistenthash.New(p.opts.Replicas, p.opts.HashFn)

	RegisterPeerPicker(func() PeerPicker { return p })
	return p
}
```

6. Set方法更新内容。先加锁，往peers写数据，然后再往httpGetters写数据。可以看到httpGetters的key是peer地址，val包含Transport和实际服务的url。
``` go
func (p *HTTPPool) Set(peers ...string) {
	p.mu.Lock()
	defer p.mu.Unlock()
	p.peers = consistenthash.New(p.opts.Replicas, p.opts.HashFn) //这里问什么又要new一次，没太看明白
	p.peers.Add(peers...)
	p.httpGetters = make(map[string]*httpGetter, len(peers))
	for _, peer := range peers {
		p.httpGetters[peer] = &httpGetter{transport: p.Transport, baseURL: peer + p.opts.BasePath}
	}
}
```

7. PickPeer是根据请求的key来返回一个ProtoGetter，这里的数据结构都在peer.go中定义过。我们只要记住这个方法是通过key来得到对应的peer即可。
``` go
func (p *HTTPPool) PickPeer(key string) (ProtoGetter, bool) {
	p.mu.Lock()
	defer p.mu.Unlock()
	if p.peers.IsEmpty() {
		return nil, false
	}
	if peer := p.peers.Get(key); peer != p.self {
		return p.httpGetters[peer], true
	}
	return nil, false
}
```

8. 这个方法是我们接收到http请求实际去执行操作。先解析请求路径，获取group和key。通过名称获取Group对象，调用他的Get方法获取返回值（至于Group的具体内部定义我们下一篇会说）。封装数据，输出。
``` go
func (p *HTTPPool) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if !strings.HasPrefix(r.URL.Path, p.opts.BasePath) {
		panic("HTTPPool serving unexpected path: " + r.URL.Path)
	}
	parts := strings.SplitN(r.URL.Path[len(p.opts.BasePath):], "/", 2)
	if len(parts) != 2 {
		http.Error(w, "bad request", http.StatusBadRequest)
		return
	}
	groupName := parts[0]
	key := parts[1]

	group := GetGroup(groupName)
	if group == nil {
		http.Error(w, "no such group: "+groupName, http.StatusNotFound)
		return
	}
	var ctx context.Context
	if p.Context != nil {
		ctx = p.Context(r)
	} else {
		ctx = r.Context()
	}

	group.Stats.ServerRequests.Add(1)
	var value []byte
	err := group.Get(ctx, key, AllocatingByteSliceSink(&value))
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	body, err := proto.Marshal(&pb.GetResponse{Value: value})
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "application/x-protobuf")
	w.Write(body)
}
```

9. httpGetter结构体封装了两个属性。
``` go
type httpGetter struct {
	transport func(context.Context) http.RoundTripper
	baseURL   string
}
```

10. sync.Pool的概念可以参看这片文章[https://www.jianshu.com/p/494cda4db297](https://www.jianshu.com/p/494cda4db297)
。大概意思就是它提供了一个池子的功能，在里面会维护一定数目的对象，这样在高并发场景下，我们可以降低内存申请的开销。如果池子中没有可用对象，会调用New方法来初始化一个。使用完后通过Put方法将对象放回池中。
``` go
var bufferPool = sync.Pool{
	New: func() interface{} { return new(bytes.Buffer) },
}
```

11. 这是我们请求其他peer获取返回值的方法。先拼接请求路径，执行http请求。返回值的处理：从bufferPool中获取一个对象，Reset将其置空，copy得到返回值，然后解析成为对象。
``` go
func (h *httpGetter) Get(ctx context.Context, in *pb.GetRequest, out *pb.GetResponse) error {
	u := fmt.Sprintf(
		"%v%v/%v",
		h.baseURL,
		url.QueryEscape(in.GetGroup()),
		url.QueryEscape(in.GetKey()),
	)
	req, err := http.NewRequest("GET", u, nil)
	if err != nil {
		return err
	}
	req = req.WithContext(ctx)
	tr := http.DefaultTransport
	if h.transport != nil {
		tr = h.transport(ctx)
	}
	res, err := tr.RoundTrip(req)
	if err != nil {
		return err
	}
	defer res.Body.Close()
	if res.StatusCode != http.StatusOK {
		return fmt.Errorf("server returned: %v", res.Status)
	}
	b := bufferPool.Get().(*bytes.Buffer)
	b.Reset()
	defer bufferPool.Put(b)
	_, err = io.Copy(b, res.Body)
	if err != nil {
		return fmt.Errorf("reading response body: %v", err)
	}
	err = proto.Unmarshal(b.Bytes(), out)
	if err != nil {
		return fmt.Errorf("decoding response body: %v", err)
	}
	return nil
}
```