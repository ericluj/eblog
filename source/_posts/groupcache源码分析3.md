---
title: groupcache源码分析3:lru.go
date: 2020-03-11 22:25:06
tags:
---

LRU算法全称Least Recently Used ，最近最少使用，当数据所占内存达到一定阈值，我们要移除掉最近最少使用的数据。

<!-- more -->

1. 定义了一个缓存结构体，MaxEntries表示缓存的数量上限，值为0不限制。OnEvicted是一个func类型，当缓存被淘汰时被调用。ll是一个双向链表指针，链表两头分别为最新和最旧的数据。cache是一个map，k是缓存名，v是链表中元素指针。
```
type Cache struct {
	MaxEntries int
	OnEvicted func(key Key, value interface{})
	ll    *list.List
	cache map[interface{}]*list.Element
}
```
2. Key是定义的类型，可包含任何对象。entry结构体，缓存的单位。
```
type Key interface{}

type entry struct {
	key   Key
	value interface{}
}
```
3. 初始化方法，创建一个lru缓存。
```
func New(maxEntries int) *Cache {
	return &Cache{
		MaxEntries: maxEntries,
		ll:         list.New(),
		cache:      make(map[interface{}]*list.Element),
	}
}
```
4. 添加方法，先惰性加载cache和ll属性，虽然new的时候会创建他俩，但是若执行clear方法会把这两个属性又置为了nil，所以这里要判断。然后判断新加的key是否已经存在，若存在将其移到ll的最前方，并返回value。否则，创建entry，将其加入ll的最前方，cache保存其在ll中的元素指针。开启了缓存限制，并且ll大小已超，调用RemoveOldest方法。
```
func (c *Cache) Add(key Key, value interface{}) {
	if c.cache == nil {
		c.cache = make(map[interface{}]*list.Element)
		c.ll = list.New()
	}
	if ee, ok := c.cache[key]; ok {
		c.ll.MoveToFront(ee)
		ee.Value.(*entry).value = value
		return
	}
	ele := c.ll.PushFront(&entry{key, value})
	c.cache[key] = ele
	if c.MaxEntries != 0 && c.ll.Len() > c.MaxEntries {
		c.RemoveOldest()
	}
}
```
5. 获取缓存，cache不存在，直接返回空。若key存在，获取其值，将其移到ll最前方。
```
func (c *Cache) Get(key Key) (value interface{}, ok bool) {
	if c.cache == nil {
		return
	}
	if ele, hit := c.cache[key]; hit {
		c.ll.MoveToFront(ele)
		return ele.Value.(*entry).value, true
	}
	return
}
```
6. removeElement，移除一个单位，分别从ll和cache中移除，若定义了OnEvicted，调用该方法。Remove包装了removeElement，为实际对外使用的方法名。RemoveOldest，获取ll最后一个元素，将其删除。
```
func (c *Cache) Remove(key Key) {
	if c.cache == nil {
		return
	}
	if ele, hit := c.cache[key]; hit {
		c.removeElement(ele)
	}
}

func (c *Cache) RemoveOldest() {
	if c.cache == nil {
		return
	}
	ele := c.ll.Back()
	if ele != nil {
		c.removeElement(ele)
	}
}

func (c *Cache) removeElement(e *list.Element) {
	c.ll.Remove(e)
	kv := e.Value.(*entry)
	delete(c.cache, kv.key)
	if c.OnEvicted != nil {
		c.OnEvicted(kv.key, kv.value)
	}
}
```
7. 获取缓存的总数。
```
func (c *Cache) Len() int {
	if c.cache == nil {
		return 0
	}
	return c.ll.Len()
}
```
8. 清理所有缓存，OnEvicted不为nil，遍历元素执行回调，然后将ll和cache置为nil，因为这个缘故所以在Add方法最开始有一个惰性加载。
```
func (c *Cache) Clear() {
	if c.OnEvicted != nil {
		for _, e := range c.cache {
			kv := e.Value.(*entry)
			c.OnEvicted(kv.key, kv.value)
		}
	}
	c.ll = nil
	c.cache = nil
}
```