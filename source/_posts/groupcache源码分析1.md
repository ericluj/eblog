---
title: groupcache源码分析1:consistenthash.go
date: 2020-03-09 22:21:58
tags: groupcache
categories: 
- go
- groupcache
---

consistenthash.go文件主要是提供一个Map数据结构，它基于一致性哈希算法，来达到根据缓存的key名，去寻找对应服务器结点的作用。

<!-- more -->

### 一致性哈希算法
参考文章：[https://www.cnblogs.com/cloudgeek/p/9427036.html](https://www.cnblogs.com/cloudgeek/p/9427036.html)

假设我们有三台机器，有一个key需要缓存到其中一台上，我们可以通过hash(key)%N的方式来取余，这样结果必定是其中的一台机器。
但是上面的方式存在一个缺点，就是如果增加或者减少结点，那么其中的缓存会被大量的转移。

一致性哈希的方式，是将存在的机器串在一起，首尾相连，看起来每台机器就是圆上的一个点。然后我们将hash(key)的值跟其最近一个点关联，存储在该台机器上，这样当该结点消失，我们只需要重新分配与其关联的缓存即可。如果添加结点，也只需要将已经存在某结点的部分缓存进行重新分配。
但是这种方式，同样存在一个缺点，如果结点较少，很容易分配不均，某台过多或者过少。

于是，我们引入虚拟结点的概念，将hash(机器+num)与机器做关联，通过缓存->虚拟结点->结点的形式进行绑定，增加了数据分配的平衡性。

### 代码分析
1. Hash定义了一个函数类型，接受[]byte返回uint32，他表示我们选择哈希运算方法类型。
``` go
type Hash func(data []byte) uint32
```
2. Map是一个数据结构，用来保存hash后的结点。
``` go
type Map struct {
	hash     Hash //哈希算法类型
	replicas int //每个机器的虚拟结点数
	keys     []int //哈希环上的一个点，其值为结点哈希算法后的返回值，升序排列
	hashMap  map[int]string //映射关系，key为keys中保存的值，val是对应机器
}
```
3. 初始化方法，返回一个Map数据结构。
``` go
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
```
4. 判断Map是否为空。
``` go
func (m *Map) IsEmpty() bool {
	return len(m.keys) == 0
}
```
5. 将机器结点添加到Map中。
``` go
func (m *Map) Add(keys ...string) {
	for _, key := range keys {
		for i := 0; i < m.replicas; i++ { //生成虚拟结点
			hash := int(m.hash([]byte(strconv.Itoa(i) + key)))
			m.keys = append(m.keys, hash)
			m.hashMap[hash] = key
		}
	}
	sort.Ints(m.keys) //排序
}
```
6. 获取结点机器的方法。
``` go
func (m *Map) Get(key string) string {
	if m.IsEmpty() { //非空判断
		return ""
	}

	hash := int(m.hash([]byte(key))) //生成key的哈希，这里key代表的是缓存数据的key
    
    //遍历[0,n),查找到符合func返回true的最小i值，若都不符合条件返回n
	idx := sort.Search(len(m.keys), func(i int) bool { return m.keys[i] >= hash })

    //这里想一下不难理解，如果是一条直线的话，我们一直取比hash值大的那个结点
    //如果这个结点不存在，则取结点0
	if idx == len(m.keys) {
		idx = 0
	}

    //返会结点对应机器
	return m.hashMap[m.keys[idx]]
}
```