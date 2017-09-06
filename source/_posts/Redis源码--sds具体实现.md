---
title: Redis源码--SDS具体实现
date: 2017-09-04 15:48:15
categories: C 
tags: [C, Redis]
---


## 前言

Redis的学习总结文章我大概分为以下四个部分:

- 数据结构
- 网络和IO
- Redis基本功能
- 其他

来进行讲述. 这是数据结构部分SDS分析的第二篇, 主要讲解SDS的具体实现, SDS类型背景及优点的讲解参见上一篇[Redis源码--sds分析](/2017/09/04/Redis源码--sds分析/).
<!--more-->

## sdshdr以及sds类型
先看`sds.h`中的相关源码:

### 源码
```C

//sds其实就是 char* 类型的指针
typedef char *sds;


//下面是5中SDS类型的定义
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
首先解释下代码中的 `__attribute__ ((__packed__))`指令的作用:

```
attribute ((packed))的用意：加上此字段是为了让编译器以紧凑模式来分配内存。如果没有这个字段，编译器会按照struct中的字段进行内存对齐，这样的话就不能保证header和sds的数据部分紧紧的相邻了，也不能按照固定的偏移来获取flags字段。
```
关于C结构体中的内存对齐问题, 请查看[Redis源码--浅谈内存对齐](/2017/05/04/Redis源码--浅谈内存对齐/).

我们可以看见, 我们使用了不同的结构体来存储SDS中的`各种size参数`和`字符串`指针.

### 内存中的状态

对于SDS在内存中的状态和C字符串有什么不同, 我们还是通过SDS初始化的函数代码来了解.

```C
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    //这个是根据 initlen 的大小来获取 sds type 的方法
    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    // 根据 sds type 来获取 sdshdr 结构体的大小
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */

    // 这里可以看出, 内存中 sdshdr 结构体和 sdshdr->buf 所指向的字符串的地址空间是连续的, 并且, 最后的 +1 是为字符串的结束标志 '\0' 分配空间
    sh = s_malloc(hdrlen+initlen+1);
    if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    if (sh == NULL) return NULL;

    //这里之所以能够使用指针直接偏移操作, 是因为上面的 __attribute__ ((__packed__))  保证了分配的内存空间是紧凑的, 没有因为内存对其而存在空闲空间
    s = (char*)sh+hdrlen; //这里的 s 指针所指向的就是 sdshdr->buf 的位置, 真正字符串的起始位置
    fp = ((unsigned char*)s)-1; // fp 指针指向的就是 sdshdr->flags的位置, 存储sds的类型信息
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }

    //当初始的字符串不为空的时候, 直接将初始字符串 copy 到 sds 的指定位置
    if (initlen && init)
        memcpy(s, init, initlen);

    //使用 '\0' 作为字符串的结束标志
    s[initlen] = '\0';

    // 注意这里返回的是 s, 即字符串的起始位置；并不是 sdshdr 结构体的起始位置
    return s;
}
```

通过上述的代码分析, 可以看出在 Redis 的系统中, 虽然重新定义了 SDS 字符串类型, 但是使用的时候, 还是 C 字符串.

### 内存中的示意图
下图展示了SDS在内存中的空间占用情况:

![SDS内存空间](/images/sds.jpg)

## sds操作函数

下面来分析一些常用的 sds 类型操作函数.

### sdsfree 释放空间

```C
void sdsfree(sds s) {
    if (s == NULL) return;
    // 将指针移动到 sdshdr 的位置, 在进行 free 操作
    // 释放掉整个 sdshdr 的内容
    s_free((char*)s-sdsHdrSize(s[-1]));
}
```

### sdsRemoveFreeSpace 清理空间

```C
//清理 SDS 中的空闲空间
sds sdsRemoveFreeSpace(sds s) {
    void *sh, *newsh;
    char type, oldtype = s[-1] & SDS_TYPE_MASK; //获取sds type 的操作
    int hdrlen;
    size_t len = sdslen(s);
    //通过 sds 指针, 获取 sdshdr 的指针
    sh = (char*)s-sdsHdrSize(oldtype);

    type = sdsReqType(len);
    hdrlen = sdsHdrSize(type);
    if (oldtype==type) {
        //type 不变的时候, realloc 减少空间
        newsh = s_realloc(sh, hdrlen+len+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        //type 改变, malloc 重新分配新空间
        newsh = s_malloc(hdrlen+len+1);
        if (newsh == NULL) return NULL;
        // copy 字符串的内容
        memcpy((char*)newsh+hdrlen, s, len+1);
        // 释放掉原来的空间
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, len);
    return s;
}
```

## sds中的帮助函数

sds的实现代码中不仅有对sds的基本操作, 还有一些有意思的帮助函数的实现, 下面我们来分析这些函数.

### long long 转换成字符串

```C
#define SDS_LLSTR_SIZE 21

//将 long long 类型转换成 string, 结果写入到 s 指针指向的地址空间,
//函数返回生成字符串的长度
int sdsll2str(char *s, long long value) {
    char *p, aux;
    unsigned long long v;
    size_t l;

    /* Generate the string representation, this method produces
     * an reversed string. */
    //这里通过将 value 不断的进行除 10 操作, 获取每一位的数字, 并将数字转换成字符,拼接在一起组成字符串
    // 这里生成的字符串是倒序, 并且包括符号
    v = (value < 0) ? -value : value;
    p = s; //p 指针指向 s
    do {
        *p++ = '0'+(v%10);
        v /= 10;
    } while(v);
    if (value < 0) *p++ = '-';

    /* Compute length and add null term. */
    l = p-s; //通过上述步骤中 p 的位移, 获取字符串的长度
    *p = '\0'; //字符串的结束标志

    /* Reverse the string. */
    p--; //最后的结束标志不用移动
    while(s < p) {
        // 现在 s 和 p 指针分别指向字符串的开头和结尾(去掉最后的'\0')
        // 通过不断的转换字符, 将倒序的字符串转换成正序
        aux = *s;
        *s = *p;
        *p = aux;
        s++;
        p--;
    }
    return l;
}
```

这个函数将 `long long` 类型的变量转换成字符串, 值得学习.

### sdscatfmt sds通过格式化输出连接字符串

```C
/* This function is similar to sdscatprintf, but much faster as it does
 * not rely on sprintf() family functions implemented by the libc that
 * are often very slow. Moreover directly handling the sds string as
 * new data is concatenated provides a performance improvement.
 *
 * However this function only handles an incompatible subset of printf-alike
 * format specifiers:
 *
 * %s - C String
 * %S - SDS string
 * %i - signed int
 * %I - 64 bit signed integer (long long, int64_t)
 * %u - unsigned int
 * %U - 64 bit unsigned integer (unsigned long long, uint64_t)
 * %% - Verbatim "%" character.
 */
sds sdscatfmt(sds s, char const *fmt, ...) {
    size_t initlen = sdslen(s);
    const char *f = fmt;
    int i;
    va_list ap;

    va_start(ap,fmt);
    f = fmt;    /* Next format specifier byte to process. */
    i = initlen; /* Position of the next byte to write to dest str. */
    while(*f) {
        char next, *str;
        size_t l;
        long long num;
        unsigned long long unum;

        /* Make sure there is always space for at least 1 char. */
        if (sdsavail(s)==0) {
            s = sdsMakeRoomFor(s,1);
        }

        switch(*f) {
            //%开始的字符串, 将它视作格式化的代码
        case '%':
            next = *(f+1);
            f++;
            //通过不同的格式获取参数, 并将其连接到字符串的尾部
            switch(next) {
            case 's':
            case 'S':
                str = va_arg(ap,char*);
                l = (next == 's') ? strlen(str) : sdslen(str);
                if (sdsavail(s) < l) {
                    s = sdsMakeRoomFor(s,l);
                }
                memcpy(s+i,str,l);
                sdsinclen(s,l);
                i += l;
                break;
            case 'i':
            case 'I':
                if (next == 'i')
                    num = va_arg(ap,int);
                else
                    num = va_arg(ap,long long);
                {
                    char buf[SDS_LLSTR_SIZE];
                    l = sdsll2str(buf,num);
                    if (sdsavail(s) < l) {
                        s = sdsMakeRoomFor(s,l);
                    }
                    memcpy(s+i,buf,l);
                    sdsinclen(s,l);
                    i += l;
                }
                break;
            case 'u':
            case 'U':
                if (next == 'u')
                    unum = va_arg(ap,unsigned int);
                else
                    unum = va_arg(ap,unsigned long long);
                {
                    char buf[SDS_LLSTR_SIZE];
                    l = sdsull2str(buf,unum);
                    if (sdsavail(s) < l) {
                        s = sdsMakeRoomFor(s,l);
                    }
                    memcpy(s+i,buf,l);
                    sdsinclen(s,l);
                    i += l;
                }
                break;
            default: /* Handle %% and generally %<unknown>. */
                s[i++] = next;
                sdsinclen(s,1);
                break;
            }
            break;
        default:
        //不是 '%' 开头的字符串, 直接连接到尾部
            s[i++] = *f;
            sdsinclen(s,1);
            break;
        }
        f++;
    }
    va_end(ap);

    /* Add null-term */
    //添加字符串结束标志
    s[i] = '\0';
    return s;
}
```

### sdssplitlen 分割字符串为数组

```C

//使用 sep 指针为分隔符, 将 s 分割成字符串数组
sds *sdssplitlen(const char *s, int len, const char *sep, int seplen, int *count) {
    //默认分割成 5 个子字符串
    int elements = 0, slots = 5, start = 0, j;
    sds *tokens;

    if (seplen < 1 || len < 0) return NULL;

    tokens = s_malloc(sizeof(sds)*slots);
    if (tokens == NULL) return NULL;

    if (len == 0) {
        *count = 0;
        return tokens;
    }


    for (j = 0; j < (len-(seplen-1)); j++) {
        /* make sure there is room for the next element and the final one */
        //根据当前已经分割的数目, 进行数组的调整
        if (slots < elements+2) {
            sds *newtokens;

            slots *= 2;
            newtokens = s_realloc(tokens,sizeof(sds)*slots);
            //一旦分配内存出错, 需要清理掉已经分配的内存, 并且结束执行
            if (newtokens == NULL) goto cleanup;
            tokens = newtokens;
        }
        /* search the separator */
        //查找分隔符, 找到了就进行分割操作
        if ((seplen == 1 && *(s+j) == sep[0]) || (memcmp(s+j,sep,seplen) == 0)) {
            tokens[elements] = sdsnewlen(s+start,j-start);
            if (tokens[elements] == NULL) goto cleanup;
            elements++;
            start = j+seplen;
            j = j+seplen-1; /* skip the separator */
        }
    }
    /* Add the final element. We are sure there is room in the tokens array. */
    tokens[elements] = sdsnewlen(s+start,len-start);
    if (tokens[elements] == NULL) goto cleanup;
    elements++;
    *count = elements;
    return tokens;

cleanup:
    {
        int i;
        for (i = 0; i < elements; i++) sdsfree(tokens[i]);
        s_free(tokens);
        *count = 0;
        return NULL;
    }
}

```

这个函数常用于分割 `Redis Command` 字符串, 以便后续处理.


## 总结

这一篇Blog主要偏重于SDS的源码解读, 主要解析了SDS实现中的主要函数, 以及背后的实现原理和中心思想. 以后的分析文章也会使用这个套路, 不仅仅是要将清楚实现的原理, `还要能整明白作者设计的意图和想要解决的问题`.



