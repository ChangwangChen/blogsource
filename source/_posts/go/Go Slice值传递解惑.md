---
title: Go Slice值传递解惑
date: 2020-07-22 13:50:45
categories: Go
tags: [Go]
---

# 问题

先上代码：

```go

package main

import "fmt"

func main() {
	s := make([]int, 0, 10)
	fmt.Println(s)
	modifyS(s)
	fmt.Println(s)
}

func modifyS(s []int) {
	s = append(s, 10)
}

//output:
//[]
//[]

```

按照 Slice 的结构体
```go
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```
来看， `SliceHeader.Data` 是指针类型， 在 `modifyS` 函数中我们 `append` 的时候应该是改变了底层数组的值， 为什么没有反映在 `main` 函数里面呢？

# 解惑

Go中没有明确使用指针的地方都是值传递(map 类型除外)， 所以在调用 `modifyS` 的时候， 参数中的 `slice` 是main函数中的copy， 所以在子函数中 append 的时候，`底层数组的值是变了， 但是main中的slice的len 和 cap 都没有改变`， 所以在main中不会体现出修改。

看下面代码的验证：

```go

package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func main() {
	s := make([]int, 1, 10)
    modifyS(s)
    //打印 s 变量的地址
	fmt.Println("main:", unsafe.Pointer(&s))

    //打印 s 底层数组的第二个值
	sh := *(*reflect.SliceHeader)(unsafe.Pointer(&s))
	i := 0
	v := *(*int)(unsafe.Pointer(sh.Data + unsafe.Sizeof(i)))
	fmt.Println("s[1]:", v)
}

func modifyS(s []int) {
    s = append(s, 11)
    //打印 s 变量的地址
	fmt.Println("sub:", unsafe.Pointer(&s))
}

//output：
//sub:  0xc00010e060
//main: 0xc00010e040
//s[1]: 11

```

可以看见 sub 和 main 中的 slice 地址不一样， 但是 mian 中也能看见底层数组的修改。


