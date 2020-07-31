---
title: Go 接口简介
date: 2020-07-27 13:50:45
categories: Go
tags: [Go]
---

# 隐式实现

Go 的接口没有在语言层面强制用户声明实现了接口， 通过 Duck Type 来隐式的实现接口。
即实现了接口中的所有方法， 就表示实现了这一接口。

# 数据结构 （两种类型）

Go 中的接口有两种类型， 包含方法的和不含方法的。数据结构分别如下：

```go
//包含方法
type iface struct {//16 bytes
	tab  *itab
	data unsafe.Pointer
}

//不包含方法
type eface struct {// 16 bytes
	_type *_type
	data  unsafe.Pointer
}

```

