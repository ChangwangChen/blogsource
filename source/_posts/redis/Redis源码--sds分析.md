---
title: Redis源码--sds分析
date: 2017-09-04 14:48:15
categories: C 
tags: [C, Redis]
---

## 前言

Redis的学习总结文章我大概分为以下四个部分:

- 数据结构
- 网络和IO
- Redis基本功能
- 其他

来进行讲述. 这是数据结构部分的第一篇, 源码解析参见下一篇[Redis源码--sds具体实现](/2017/09/04/Redis源码--sds具体实现/).
<!--more-->

## 为什么要有SDS

首先说明, SDS是`simple dynamic strings`的缩写, 其实在名称中能看出来这种数据结构是对C语言字符串的封装.其底层是使用C语言自带的字符串实现的, 但是规避了一些问题.下面来简述下SDS的优点.

### 记录size

SDS会记录字符串的`内存分配大小`和`字符串已使用的大小`.这样在使用 `sdslen()`,`sdsavail()`和`sdsalloc()`来获取字符串长度, 字符串可用长度和字符串的内存分配长度的时候, 是O(1)的时间复杂度, 而不是C字符串里面 O(n)的时间复杂度.

### 防止缓冲区溢出

接上一条内容, 由于SDS会记录字符串的size信息, 在对SDS进行赋值,连接等等操作的时候, 可以很方便的检查内存是否够用从而重新分配合适的内存空间,这样能够有效的防止字符串缓冲区的内存溢出问题.

### 减少API操作的内存分配次数

接上两条内容, SDS记录了当前分配的内存空间大小和使用空间大小, 以及空余的空间大小, 所以在进行操作的时候, 会首先检查空间大小是否足够, 只有在不够使用的时候才会重新分配内存空间.

- 增加分配内存空间的时候, 会采取其他的策略来进行空间的预分配；在清空SDS的时候
- 减少size的时候, 不会直接将SDS的内存空间缩减到指定大小, 防止在增加空间的时候, 进行不必要的两次内存分配工作
- 删除SDS的时候,不会直接`free`内存空间, 而是标记当前的SDS为空, 真正删除的时候才会进行`free`操作；.

```C
//SDS内存空间预分配策略
#define SDS_MAX_PREALLOC (1024*1024)  //1MB

    //新的 size 小于 1MB, 直接分配 2 倍的长度
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
    //新的 size 大于或者等于 1MB, 直接预分配 1MB 的空间
        newlen += SDS_MAX_PREALLOC;

```

SDS分为`5种实现方式`:`sdshdr5`, `sdshdr8`, `sdshdr16`, `sdshdr32`和`sdshdr64`.
通过下面的函数可以知道各种类型的字符串的`最大size`.

```C
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5)
        return SDS_TYPE_5;
    if (string_size < 1<<8)
        return SDS_TYPE_8;
    if (string_size < 1<<16)
        return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX)
    if (string_size < 1ll<<32)
        return SDS_TYPE_32;
#endif
    return SDS_TYPE_64;
}
```

`这个是 `SDS2.0`版本里面更新的, 为什么使用这样的类型划分呢???`

我们来看下面的代码: 
```C
sds sdsRemoveFreeSpace(sds s) {
    void *sh, *newsh;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    size_t len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);

    type = sdsReqType(len);
    hdrlen = sdsHdrSize(type);
    if (oldtype==type) {
        //SDS的类型没有变化, 直接使用 realloc 方法进行扩展内存空间
        newsh = s_realloc(sh, hdrlen+len+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        //SDS类型发生变化, 使用 malloc 重新分配内存, 并且将 s 所指向的字符串 copy 到新分配内存空间的指定位置
        newsh = s_malloc(hdrlen+len+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, len);
    return s;
}
```

这样看来, 新版本的SDS实现中引入了`type`的概念, 是为了在操作的时候, 减少内存复制操作.

### 二进制安全

```
    其实我一直不太明白什么是二进制安全, 我能给出的解释就是, 在对字符串进行函数操作的时候, 不会丢失信息.
```

众所周知, C语言的字符串本身不是二进制安全的, 因为C语言的字符串是使用`\0`作为字符串的结束标志, 这样在字符串中就不能出现`\0`字符.例如, `strlen("Redis\0SDS")`, 就会返回`5`, 而不是`8`.这样, 对字符串进行操作的时候, 就会产生错误.

SDS虽然也使用`\0`作为字符串的结尾, 但是如上面所说, 他记录了字符串的`各种size`, 这样进行字符串的操作之前会进行`size`的比较, 保证不会出现上述情况.

由于有`size`信息的加持, 所以SDS中不仅能够保存字符信息, 也可以保存所有的二进制信息, 我们在对其进行操作的时候, 会根据`size`的指示进行, 不会出现任何问题.


### 兼容C字符串的规则

上一条中有提到SDS也使用`\0`作为字符串的结束标志, 而且在后面讲述SDS的结构的时候, 也会讲到SDS的API操作的是`直接指向字符串的指针`, 保证了其本质和C语言的字符串一致.

这样对SDS类型进行操作的时候, 会复用一些 `string.h` 里面的C字符串库函数进行, 方便快捷.

##  总结

这一篇Blog只要是讲述了`Redis数据结构SDS`的一些背景知识, 和它试图解决的问题, 以及这样做的优点. SDS结构的具体实现分析, 我们将会放在下一篇[Redis源码--sds具体实现](/2017/09/04/Redis源码--sds具体实现/)中讲解.



