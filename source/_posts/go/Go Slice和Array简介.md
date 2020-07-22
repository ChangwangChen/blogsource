---
title: Go Slice和Array简介
date: 2020-07-13 13:50:45
categories: Go
tags: [Go]
---

# Array

数组在Go语言中和在其他语言中的本质没有区别， 就是一块连续的内存块，访问的时候直接通过下标找到对应的元素信息。数组还有一个重要的功能就是做为 `slice` 的底层数据载体。

另外我们需要知道的一点是 Go 中的数组是`值传递`， 不管是赋值还是传参，都会 copy 一份新的数组。看代码：

```go

    func main() {
        a := [7]int{1,2,3,4,5,6,7}
        var a1 [7]int
        a1 = a
        fmt.Printf("&a: %p, &a1: %p\n", &a, &a1)
        
        show(a)
    }

    func show(a [7]int) {
        fmt.Printf("func arg &a: %p", &a)
    }

    
    //输出
    &a: 0xc00001c1c0, &a1: 0xc00001c200
    func arg &a: 0xc00001c240

```

3 个数组的地址都不一样， 可见 Go 中数组是值传递。 所以在程序中大数组参数在传递的时候需要考虑 copy 的问题， 我们可以通过传递 `数组指针` 来避免复制的问题。

# Slice

## 数据结构

切片数据结构在Go中`运行时`是通过`SliceHeader`来表示：

```go

type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}

/*
Data 作为一个指针指向的数组是一片连续的内存空间，这片内存空间可以用于存储切片中保存的全部元素，数组中的元素只是逻辑上的概念，底层存储其实都是连续的，所以我们可以将切片理解成一片连续的内存空间加上长度与容量的标识。
*/

```

## 访问和赋值

Slice使用下标进行访问元素，这个和数组类似。也可以通过 `len(s)` 和 `cap(s)` 来访问 Slice 的长度和容量。

上面展示的Slice数据结构可以看出， 可能会有多个Slice底层引用的是同一地址，所以在使用Slice的时候需要特别注意这个问题。

## 追加和扩容

Go通过 `append()` 向Slice中追加元素

```go
//append函数原型
func append(slice []Type, elems ...Type) []Type

```
 append处理流程分为两个分支：

* len(s) + len(elems)  <= cap(s), 不会产生扩容， 但是会修改底层数组
* len(s) + len(elems)  > cap(s), 会产生扩容， elems 的值不会出现在底层数组

Slice 通过 `growslice` 来进行扩容，基本的思路就是

```go

func growslice(et *_type, old slice, cap int) slice {
	newcap := old.cap
    doublecap := newcap + newcap
    //期望的 cap  > 2*newcap, 直接使用期望的 cap
	if cap > doublecap {
		newcap = cap
	} else {
        //当前切片的大小 < 1024, 直接 *2
		if old.len < 1024 {
			newcap = doublecap
		} else {
            //0 < newcap 防止溢出
            //每次都是 1.25 倍的增长， 知道超过期望的 cap
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

```
扩容总结：

1. 如果期望容量大于当前容量的两倍就会使用期望容量；
2. 如果当前切片长度小于 1024 就会将容量翻倍；
3. 如果当前切片长度大于 1024 就会每次增加 25% 的容量，直到新容量大于期望容量；


其实扩容到这里并没有结束， 为了配合Go中内存分配的需要， 系统会计算slice扩容之后占用字节数然后分别进行内存对齐， 最终会计算出来 cap 的值。