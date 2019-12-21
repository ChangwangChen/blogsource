---
title: Opensuse系统源码安装PHP7
date: 2016-09-22 10:52:33
categories: PHP
tags: [PHP,Opensuse]
---

# 为什么

工作的时候一直使用的是Opensuse操作系统，自从使用了 Laravel 框架之后，系统自带的 PHP 版本显得有点落后，就萌生了自己编译安装 PHP 7.0 的念头。

<!--more-->

# 怎么做

## 下载源码包

可以去 [PHP官网](http://www.php.net) 或者 [Github地址](https://github.com/php/php-src/releases) 下载指定版本的PHP源码。

## 编译

其实Linux系统源码编译安装软件是有一定的规范的，（`个人观点，可能不正确`）无非是三个步骤：configure、make、make install。

首先解压下载的源码包并进入文件夹中，在命令行中输入指令 

``` 
 ./configure --help
```

查看 `configure` 的帮助文档。

对于 PHP 的 configure 来说，主要有这几种配置选项：

```
--prefix  //软件的安装目录

--*dir     //软件不同文件的存放目录，包括可执行文件、配置文件、库文件、文档文件等等

--enable-* 
--disable-*
--with-*
--without-*

//这些是对PHP不同模块的配置信息，具体的细节可输入上面的指令自己查看
```

对于我自己来说，PHP的组建基本都安装了，下面是我的 configure 配置信息：

```
./configure  \
  --prefix=/usr/local/php7 \
  --bindir=/usr/bin \
  --sbindir=/usr/sbin \
  --libdir=/usr/lib64/php \
  --localstatedir=/var/php7 \
  --includedir=/usr/include/php7 \
  --sysconfdir=/etc/php7  \
  --enable-fpm \
  --with-fpm-systemd \
  --with-config-file-path=/etc/php7/ \
  --with-openssl \
  --with-curl=/usr/include/curl \
  --with-zlib \
  --with-libxml=/usr/include/libxml \
  --with-gd=/usr/include/ \
  --enable-ftp  \
  --with-freetype-dir=/usr/include/freetype \
  --with-gettext=/usr/include/ \
  --enable-intl \
  --enable-mbstring \
  --with-mcrypt=/usr/include/ \
  --with-mysqli=/usr/bin/mysql_config \
  --enable-pcntl \
  --with-pdo-mysql \
  --enable-soap \
  --enable-sockets \
  --enable-mysqlnd \
  --enable-zend-signals \
  --with-xpm-dir \
  --enable-bcmath \
  --enable-zip

```

## 安装

接下来就进行 `make` 和 `make install` 指令。在 `make` 指令的时候可能会报错，这主要是因为 PHP 的某些模块依赖的其他软件没有安装导致的，我们按照使用 `zypper` 安装即可。

这里安装其他依赖软件的时候，大部分是需要安装源码的，可使用 `zypper install *-devel` 来安装。


# 配置自启动

这里主要讲的是 `php-fpm` 的开机自启动设置，我的方法是在 `/etc/init.d/after.local` 文件中加入 `php-fpm`。

这里只是我自己的方法，能用，但是可能不够好。
