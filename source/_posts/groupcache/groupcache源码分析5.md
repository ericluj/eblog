---
title: groupcache源码分析5:byteview.go
date: 2020-03-15 21:31:42
tags: groupcache
categories: 
- go
- groupcache
---

byteview.go文件提供了一个数据结构ByteView，它是对[]byte或string类型的一个封装，提供了一些方法。

<!-- more -->

1. 结构体定义，含有[]byte类型b和string类型s，优先使用b，b为nil再使用s。
``` go
type ByteView struct {
	b []byte
	s string
}
```

2. 返回长度。
``` go
func (v ByteView) Len() int {
	if v.b != nil {
		return len(v.b)
	}
	return len(v.s)
}
```

3. 基于结构体内容，返回一个[]byte类型。
``` go
func (v ByteView) ByteSlice() []byte {
	if v.b != nil {
		return cloneBytes(v.b)
	}
	return []byte(v.s)
}
```

4. 基于结构体内容，返回一个string类型。
``` go
func (v ByteView) String() string {
	if v.b != nil {
		return string(v.b)
	}
	return v.s
}
```

5. 返回数据中指定位置的原始字节。**这里有一个小细节值得注意，那就是v.s[i]返回的是原始字节，而并不是如我们所想的返回字符串中某一个字符。**
``` go
func (v ByteView) At(i int) byte {
	if v.b != nil {
		return v.b[i]
	}
	return v.s[i]
}
```

6. 截取数据，返回的仍然是一个ByteView。
``` go
func (v ByteView) Slice(from, to int) ByteView {
	if v.b != nil {
		return ByteView{b: v.b[from:to]}
	}
	return ByteView{s: v.s[from:to]}
}
```

7. 截取指定索引位置到结尾的数据，返回ByteView。
``` go
func (v ByteView) SliceFrom(from int) ByteView {
	if v.b != nil {
		return ByteView{b: v.b[from:]}
	}
	return ByteView{s: v.s[from:]}
}
```

8. 拷贝数据到一个[]byte变量中。
``` go
func (v ByteView) Copy(dest []byte) int {
	if v.b != nil {
		return copy(dest, v.b)
	}
	return copy(dest, v.s)
}
```

9. 两个ByteView做比较。
``` go
func (v ByteView) Equal(b2 ByteView) bool {
	if b2.b == nil {
		return v.EqualString(b2.s)
	}
	return v.EqualBytes(b2.b)
}
```

10. 与字符串做比较，如果v的b为空，那么直接拿string类型s判断是否相等即可。否则，需要拿[]byte类型b和string比较，先判断长度是否相等，再逐个字节比对。
``` go
func (v ByteView) EqualString(s string) bool {
	if v.b == nil {
		return v.s == s
	}
	l := v.Len()
	if len(s) != l {
		return false
	}
	for i, bi := range v.b {
		if bi != s[i] {
			return false
		}
	}
	return true
}
```

11. 与上一个方法类似，只不过是与[]byte的比较。
``` go
func (v ByteView) EqualBytes(b2 []byte) bool {
	if v.b != nil {
		return bytes.Equal(v.b, b2)
	}
	l := v.Len()
	if len(b2) != l {
		return false
	}
	for i, bi := range b2 {
		if bi != v.s[i] {
			return false
		}
	}
	return true
}
```

12. 调用NewReader方法返回io.ReadSeeker类型。
``` go
func (v ByteView) Reader() io.ReadSeeker {
	if v.b != nil {
		return bytes.NewReader(v.b)
	}
	return strings.NewReader(v.s)
}
```

13. 从ByteView指定位置读取数据拷贝到[]byte中。
``` go
func (v ByteView) ReadAt(p []byte, off int64) (n int, err error) {
	if off < 0 {
		return 0, errors.New("view: invalid offset")
	}
	if off >= int64(v.Len()) {
		return 0, io.EOF
	}
	n = v.SliceFrom(int(off)).Copy(p)
	if n < len(p) {
		err = io.EOF
	}
	return
}
```

14. 读取数据写入到ByteView中。
``` go
func (v ByteView) WriteTo(w io.Writer) (n int64, err error) {
	var m int
	if v.b != nil {
		m, err = w.Write(v.b)
	} else {
		m, err = io.WriteString(w, v.s)
	}
	if err == nil && m < v.Len() {
		err = io.ErrShortWrite
	}
	n = int64(m)
	return
}
```