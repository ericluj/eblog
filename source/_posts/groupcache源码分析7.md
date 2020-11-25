---
title: groupcache源码分析7
date: 2020-03-22 21:34:30
tags: groupcache
categories: 
- go
- groupcache
---

peers.go文件的主要作用是用来定义如何查找和与其他peer沟通，peer的中文翻译为同伴，可以理解为其他机器。

<!-- more -->

1. ProtoGetter必须被一个peer实现，他有一个Get方法用来获取返回值。PeerPicker接口是用来根据key寻找peer，如果返回nil,false，那么说明，该缓存的拥有者为自身。
``` go
type ProtoGetter interface {
	Get(ctx context.Context, in *pb.GetRequest, out *pb.GetResponse) error
}

type PeerPicker interface {
	PickPeer(key string) (peer ProtoGetter, ok bool)
}
```

2. NoPeers结构体，表示没有找到peer。
``` go
type NoPeers struct{}

func (NoPeers) PickPeer(key string) (peer ProtoGetter, ok bool) { return }
```

3. 变量portPicker是一个func，返回值为PeerPicker，只能被定义一次。RegisterPeerPicker和RegisterPerGroupPeerPicker两者之一会被调用，调用时机为第一个group创建时，目的是为portPicker赋值。
``` go
var (
	portPicker func(groupName string) PeerPicker
)

func RegisterPeerPicker(fn func() PeerPicker) {
	if portPicker != nil {
		panic("RegisterPeerPicker called more than once")
	}
	portPicker = func(_ string) PeerPicker { return fn() }
}

func RegisterPerGroupPeerPicker(fn func(groupName string) PeerPicker) {
	if portPicker != nil {
		panic("RegisterPeerPicker called more than once")
	}
	portPicker = fn
}
```

4. getPeers用来返回注册的PeerPicker方法。
``` go
func getPeers(groupName string) PeerPicker {
	if portPicker == nil {
		return NoPeers{}
	}
	pk := portPicker(groupName)
	if pk == nil {
		pk = NoPeers{}
	}
	return pk
}
```