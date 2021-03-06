---
title: Go 字符串简介
date: 2020-07-13 13:50:45
categories: Go
tags: [Go]
---

# 底层数据结构
字符串在 Go 中是以下面的形式提供的:

```go
    //底层数据结构
    type _string struct {
        elements *byte // underlying bytes
        len      int   // number of bytes
    }

    //运行时查看
    s := "changwang"
    sh := *(*reflect.StringHeader)(unsafe.Pointer(&s))
    fmt.Println(sh) //{Data:4990522 Len:9}
```

# 编码

在Go中，所有的字符串常量都被视为是UTF-8编码的。 在编译时刻，非法UTF-8编码的字符串常量将导致编译失败。 在运行时刻，Go运行时无法阻止一个字符串是非法UTF-8编码的。

Go 中使用 `rune(int32)` 类型来表示Unicode编码的码点， 注意这与 `byte(uint8)` 类型不同， 所以在 Go 中处理字符串通常有 2 中模式。

```go

	s := "éक्षिaπ囧"
	for i, rn := range s {
		fmt.Printf("%2v: 0x%x %v \n", i, rn, string(rn))
	}

	fmt.Println()
	for i := 0; i < len(s); i++ {
		fmt.Printf("%2v: 0x%x %v \n", i, s[i], string(s[i]))
	}
    fmt.Println(len(s))
    
    //输出：
     0: 0x65 e 
     1: 0x301 ́ 
     3: 0x915 क 
     6: 0x94d ् 
     9: 0x937 ष 
    12: 0x93f ि 
    15: 0x61 a 
    16: 0x3c0 π 
    18: 0x56e7 囧 

     0: 0x65 e 
     1: 0xcc Ì 
     2: 0x81   
     3: 0xe0 à 
     4: 0xa4 ¤ 
     5: 0x95   
     6: 0xe0 à 
     7: 0xa5 ¥ 
     8: 0x8d   
     9: 0xe0 à 
    10: 0xa4 ¤ 
    11: 0xb7 · 
    12: 0xe0 à 
    13: 0xa4 ¤ 
    14: 0xbf ¿ 
    15: 0x61 a 
    16: 0xcf Ï 
    17: 0x80   
    18: 0xe5 å 
    19: 0x9b   
    20: 0xa7 § 
    21
```
`len(s)`获取的是`字节数`， `for...range`遍历的是`[]rune`。

# 特性 immutable 和 comparable

## immutable

字符串的底层字节数组是`immutable`的， 我们可以将其看作是`只读的[]byte`。因此我们不能通过下标对字符串进行赋值和取址操作。

由于原始字符串的不可变， 但是 `[]byte` 类型是可以任意操作的， 所以在这两者期间进行转换的时候， 会出现`内存的copy`。我们看下底层的代码：

```go

//[]byte => string
func slicebytetostring(buf *tmpBuf, b []byte) (str string) {
	l := len(b)
	if l == 0 {
		return ""
	}
	if l == 1 {
		stringStructOf(&str).str = unsafe.Pointer(&staticbytes[b[0]])
		stringStructOf(&str).len = 1
		return
	}
	var p unsafe.Pointer
	if buf != nil && len(b) <= len(buf) {
		p = unsafe.Pointer(buf)
	} else {
		p = mallocgc(uintptr(len(b)), nil, false)
	}
	stringStructOf(&str).str = p
    stringStructOf(&str).len = len(b)
    //copy 
	memmove(p, (*(*slice)(unsafe.Pointer(&b))).array, uintptr(len(b)))
	return
}

//string => []byte
func stringtoslicebyte(buf *tmpBuf, s string) []byte {
	var b []byte
	if buf != nil && len(s) <= len(buf) {
		*buf = tmpBuf{}
		b = buf[:len(s)]
	} else {
		b = rawbyteslice(len(s))
    }
    //copy
	copy(b, s)
	return b
}

```
也有一些特殊情况下， Go的编译器会避免内存copy：

* `for...range` 中代码显式的转换
* 生成 `map` key的时候
* 字符串比较时候的显式转换
* 对已存在非空字符串进行拼接时候的显式转换

通过代码来查看：

```go

sb := []byte{'c', 'h', 'a', 'n', 'g'}
s  := "chang"
m  := make(map[string]string)

//1
for _, b := range []byte(s) {
    ...
}

//2
m[string([]sb)] = s

//3
if s == string(sb) {
    ...
}

//4
ss := s + string(sb)
 

```

另外， `fasthttp`中为了避免转换带来的内存copy开销， 使用了以下代码来进行优化：

```go

// b2s converts byte slice to a string without memory allocation.
// See https://groups.google.com/forum/#!msg/Golang-Nuts/ENgbUzYvCuU/90yGx7GUAgAJ .
//
// Note it may break if string and/or slice header will change
// in the future go versions.
func b2s(b []byte) string {
	/* #nosec G103 */
	return *(*string)(unsafe.Pointer(&b))
}

// s2b converts string to a byte slice without memory allocation.
//
// Note it may break if string and/or slice header will change
// in the future go versions.
func s2b(s string) (b []byte) {
	/* #nosec G103 */
	bh := (*reflect.SliceHeader)(unsafe.Pointer(&b))
	/* #nosec G103 */
	sh := *(*reflect.StringHeader)(unsafe.Pointer(&s))
	bh.Data = sh.Data
	bh.Len = sh.Len
	bh.Cap = sh.Len
	return b
}

```

## comparable

Go在语言层面直接支持了字符串的比较， 我们可以通过 `>`、`>=`、`<`、`<=`、`==` 和 `!=` 运算符对字符串进行比较。这里需要注意：

1. 字符串比较是对底层字节数组的逐个字节进行比较
2. 如果字符串之另一字符串的前缀， 并且另一字符串较长， 最另一字符串较大
3. `==` 和 `!=` 有优化， 直接比较长度或者底层字符串的地址

代码：

```go

	bs := make([]byte, 1<<26)
	s0 := string(bs)
	s1 := string(bs)
	s2 := s1

	// s0、s1和s2是三个相等的字符串。
	// s0的底层字节序列是bs的一个深复制。
	// s1的底层字节序列也是bs的一个深复制。
	// s0和s1底层字节序列为两个不同的字节序列。
	// s2和s1共享同一个底层字节序列。

	startTime := time.Now()
	_ = s0 == s1
	duration := time.Now().Sub(startTime)
	fmt.Println("duration for (s0 == s1):", duration)

	startTime = time.Now()
	_ = s1 == s2
	duration = time.Now().Sub(startTime)
	fmt.Println("duration for (s1 == s2):", duration)

```

输出：

```
duration for (s0 == s1): 10.462075ms
duration for (s1 == s2): 136ns

```
