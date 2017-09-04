---
title: C语言中static和extern浅谈
date: 2017-09-04 13:50:45
categories: C
tags: [C]
---

## 写在前面

C语言中的 static 和 extern 关键字都是作用在变量和函数中的, 所以我们会通过变量和函数来分别进行阐述.

<!--more-->

首先我们应该明白几个问题, 关于C语言中的声明和定义:

```
1. 函数和变量的声明不会分配内存, 但是定义会分配相应的内存空间
2. 函数和变量的声明可以有很多次, 但是定义最多只能有一次
3. 函数的声明和定义方式默认都是 extern 的, 即函数默认是全局的
4. 变量的声明和定义方式默认都是局部的, 在当前编译单元或者文件内可用
```


## 关于函数

前面说了函数的声明和定义默认都是 `extern` 的, 即全局可见的. 所以在声明和定义的时候, 默认都是 `extern`.
下面的代码:

```C
int add(int a, int b);
```

在编译器看来, 和下面的代码等价:

```C
extern int add(int a, int b);
```

因此, 为了修饰当前文件中的内部函数, static 关键字出场了. 下面的例子:

```C
static int add(int a, int b);
```

`static`修饰的 `function` 表明这个函数`只对于当前文件中的其他函数是可见的, 其他文件中的函数不能够调用`.

## 关于变量

使用 `static` 和 `extern` 修饰变量的时候, 变量的生命周期是一样的, 不同的是变量的作用域.

一个表格说明关系:

关键字 | 生命周期 | 作用域
------- | ------- | -------
 extern | 静态（程序结束后释放） | 外部（整个程序）
 static	| 静态（程序结束后释放） | 内部（仅编译单元，一般指单个源文件）
 auto,register | 函数调用（调用结束后释放） |  无



## 参考文献
1. [Understanding “extern” keyword in C](http://www.geeksforgeeks.org/understanding-extern-keyword-in-c/)
2. [静态变量](http://zh.wikipedia.org/wiki/%25E9%259D%2599%25E6%2580%2581%25E5%258F%2598%25E9%2587%258F)