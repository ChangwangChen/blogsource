---
title: Redis源码--浅谈内存对齐
date: 2017-05-04 14:48:15
categories: C 
tags: [C, Redis]
---

 ```
 以下部分只是作者的臆测,
 还没有的到证实,
 因为作者的C语言水平有限,还望有高手能够出来指导,
 谢谢..
 ```

# 疑惑1
最近在“研究”Redis的源码，刚看到内存管理`[zmalloc]`的部分，发现其中的一段代码很是奇怪(可能在我看来很奇怪)，代码如下：

```c
#define update_zmalloc_stat_alloc(__n) do { \
    size_t _n = (__n); \
    //重点在这里
    if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); \
    if (zmalloc_thread_safe) { \
        update_zmalloc_stat_add(_n); \
    } else { \
        used_memory += _n; \
    } \
} while(0)
```

这个函数的功能是每次Redis申请内存的时候，更新下Redis所申请的内存总数`used_memory`.但是其中一行代码:

```c
if (_n&(sizeof(long)-1)) 
    _n += sizeof(long)-(_n(sizeof(long)-1));
```
它的意思是将申请的内存字节数目对于`sizeof(long)`来向上取整,即总是取`sizeof(long)`的倍数.

## 解释(机器字长)
CPU在读取内存的时候, 并不是按照一个字节一个字节的来读取, 而是一次读取一个`机器字`的长度. 例如 32 位的CPU一次读取 32 bits的数据, 也就是 `32 / 8 = 4 Bytes(4个字节)`的内存信息.

zmalloc 中这样来处理分配内存的大小, 是为了在为同一个数据分配的内存, 能够在最少的 CPU 时钟周期里读取完成.

# 疑惑2
同样是分配内存空间的代码中, 每次分配内存的时候会在内存的起始位置加上 `PREFIX_SIZE` 大小的内存, 并且存放的是此次`分配内存的大小`.

`PREFIX_SIZE`的定义如下:

```C
#ifdef HAVE_MALLOC_SIZE
#define PREFIX_SIZE (0)
#else
#if defined(__sun) || defined(__sparc) || defined(__sparc__)
#define PREFIX_SIZE (sizeof(long long))
#else
#define PREFIX_SIZE (sizeof(size_t))
#endif
#endif
```

不同的OS结构下面, 其值不一样.

对于分配内存的时候, 多分配 `PREFIX_SIZE` 字节的内存空间并且存储此次分配内存空间的大小, 代码如下:

```C
void *zmalloc(size_t size) {
    //这里可见分内的时候, 会多分配 PREFIX_SIZE 字节
    void *ptr = malloc(size + PREFIX_SIZE);

    if(!ptr)
        zmalloc_oom_handler(size);

#ifdef HAVE_MALLOC_SIZE
    //这里表示 OS 会自己统计已分配内存的大小
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
#else
    *((size_t*)ptr) = size; //转换指针 ptr 的类型, 并且在指针开始的 PREFIX_SIZE 个字节中赋值 size
    update_zmalloc_stat_alloc(size + PREFIX_SIZE);
    // 转换指针 ptr 的类型为 char* , 操作指针偏移的时候, 便宜量保证只有一个字节
    //
    // 通过之前的代码可以看见, 在申请内存的时候总是会多申请 PREFIX_SIZE 个字节的空间,
    // 并且在这些空间里面保存了申请内存空间的大小 size
    //

    //可见这里返回的指针已经向右偏移了 PREFIX_SIZE 个字节, 也就是实际的内存开始位置
    return (char*)ptr + PREFIX_SIZE;
#endif
}
```

## 解释

这样做的的目的是为了能够快速的获取指针指向变量的内存空间的大小.
例如, 在获取字符串所占的内存空间的时候, 我们只需要将指针左移 `PREFIX_SIZE`字节, 然后获取里面 size 的值即可, 时间复杂度是常数级别 O(1).

# 最后
这里附加一张通过 zmalloc 分配内存之后的内存图, 当然图片是从网上找的. 点击[这里](http://www.voidcn.com/blog/caishenfans/article/p-616378.html)查看原图.

![zmalloc](/images/zmalloc.jpg)

# 附:

* [Redis中的内存管理:关于zmalloc](http://www.voidcn.com/blog/caishenfans/article/p-616378.html)
