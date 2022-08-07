---
title: groupcache源码分析6:sink.go
date: 2020-03-21 21:33:20
tags: groupcache
categories: 
- go
- groupcache
---

sink.go文件实际上也是提供数据结构，在ByteView之上做了一层封装。

<!-- more -->

1. Sink接口，提供三个set方法和一个view方法，view方法返回ByteView。
``` go
type Sink interface {
	SetString(s string) error

	SetBytes(v []byte) error

	SetProto(m proto.Message) error

	view() (ByteView, error)
}
```

2. []byte类型拷贝。
``` go
func cloneBytes(b []byte) []byte {
	c := make([]byte, len(b))
	copy(c, b)
	return c
}
```

3. setSinkView方法，将一个ByteView内容设置到Sink中。注意这里viewSetter的使用，通过它去断言s是否包含setView方法。
``` go
func setSinkView(s Sink, v ByteView) error {
	type viewSetter interface {
		setView(v ByteView) error
	}
	if vs, ok := s.(viewSetter); ok {
		return vs.setView(v)
	}
	if v.b != nil {
		return s.SetBytes(v.b)
	}
	return s.SetString(v.s)
}
```

4. 文件中定义了不少实现Sink接口的结构体，但是代码中实际只用到了allocBytesSink。我们这里只单独分析一下它。创建方法和allocBytesSink结构体定义。
``` go
func AllocatingByteSliceSink(dst *[]byte) Sink {
	return &allocBytesSink{dst: dst}
}

type allocBytesSink struct {
	dst *[]byte //这里有一个疑问，此属性有什么用呢？看代码它还是一个新分配地址，但是view方法并不需要它啊。
	v   ByteView
}
```

5. view方法没啥可说的。值得注意得是另一个setView方法，我们可以使用第2步中的setSinkView方法来赋值。还有一个有意思的点，如果v存储了[]byte，那么cloneBytes方法创建出了一个新的切片赋值，否则通过string转[]byte生成一个新的切片赋值(一个新知识点：[]byte的变量赋值指向同一个地址，string和[]byte的转换是数据复制)。
``` go
func (s *allocBytesSink) view() (ByteView, error) {
	return s.v, nil
}

func (s *allocBytesSink) setView(v ByteView) error {
	if v.b != nil {
		*s.dst = cloneBytes(v.b)
	} else {
		*s.dst = []byte(v.s)
	}
	s.v = v
	return nil
}
```

6. 使用[]byte设置s和v
``` go
func (s *allocBytesSink) setBytesOwned(b []byte) error {
	if s.dst == nil {
		return errors.New("nil AllocatingByteSliceSink *[]byte dst")
	}
	*s.dst = cloneBytes(b) // clone后赋值，如果b被更改，这里的dst不会变
	s.v.b = b
	s.v.s = ""
	return nil
}
```

7. 通过proto的Message类型和[]byte类型进行赋值，内部均调用了上面setBytesOwned方法
``` go
func (s *allocBytesSink) SetProto(m proto.Message) error {
	b, err := proto.Marshal(m)
	if err != nil {
		return err
	}
	return s.setBytesOwned(b)
}

func (s *allocBytesSink) SetBytes(b []byte) error {
	return s.setBytesOwned(cloneBytes(b))
}
```

8. string类型的赋值
``` go
func (s *allocBytesSink) SetString(v string) error {
	if s.dst == nil {
		return errors.New("nil AllocatingByteSliceSink *[]byte dst")
	}
	*s.dst = []byte(v)
	s.v.b = nil
	s.v.s = v
	return nil
}
```