---
title: [译] SOCKET man 页面翻译
date: 2020-08-03 13:50:45
categories: network
tags: [Network]
---

# 名称

socket - Linux socket 接口

# 简介

```c
#include <sys/socket.h>
sockfd = socket(int socket_family, int socket_type, int protocol);
```

# 描述

此手册描述 Linux 网络 socket 层的用户接口信息. BSD 兼容的 sockets 是用户程序和内核网络协议栈的统一接口. 协议模块被划分为不同的分组, 例如 `protocol family` 中的 `AF_INET`, `AF_IPX` 和 `AF_PACKET`； `socket_type` 中的 `SOCK_STREAM` 和 `SOCK_DGRAM`. 查看 socket(2) 获取更多消息.

## Socket 层方法

这些方法被用户程序用来发送或者接收数据包或者其他功能.查看它们自己的手册获取更多信息.

`socket(2)` 创建一个 socket, `connect(2)` 链接本地的 socket 到远端的另一个 socket 上, `bind(2)`  将 socket 绑定到一个本地 socket 地址, `listen(2)` 告诉 socket 需要接受新的链接, `accept(2)` 用来获取 socket 上面新的接入链接. `socketpair(2)` 返回已经接通的两个匿名 socket(只被一些本地 `socket family` 实现, 例如: `AF_UNIX`).

`send(2)`, `sendto(2)`, `sendmsg(2)` 发送数据；`recv(2)`, `recvfrom(2)`, `recvmsg(2)` 接收数据. `poll(2)` 和 `select(2)` 等待数据到达或者准备发送数据. 另外, 标准 IO 操作例如 `write(2)`, `writev(2)`, `sendfile(2)`, `read(2)` 和 `readv(2)` 也能用于从 socket 读取或者写入数据.

`getsockname` 返回本地 socket 的地址, `getpeername` 返回远端 socket 的地址. `getsockopt` 和 `setsockopt` 用来获取和设置 socket 层或者协议层配置信息. `ioctl` 可以用来设置或者获取其他信息.

`close` 用来关闭链接. `shutdown` 则用来关闭全双工链接的部分功能(`SHUT_RD`, `SHUT_WR` 和 `SHUT_RDWR`).

查找, 或者在非零位置调用 `pread` 或者 `pwrite` 都不被支持.

可以使用 `fcntl` 对 socket 的文件描述符使用 `O_NONBLOCK` 标志来开启 socket 的非阻塞 IO 读写. 那么所有的可能阻塞的操作就会返回 `EAGAIN`(稍后重试)错误；`connect` 会返回 `EINPROGESS` 错误. 这是有用户可以等待 `poll` 或者 `select` 返回事件.

|  | IO 事件 | |
| :---: | :--- | :---- |
| 事件 | Poll 标志 | 触发时机 |
| Read | POLLIN | 新数据到达 |
| Read | POLLIN | 新的链接已经建立(对于面向链接的 socket) |
| Read | POLLHUP | 对方发起了链接关闭 |
| Read | POLLHUP | 链接已经断开(只针对面向链接的 socket). 当 socket 被写入了 SIGPIPE 信号也会同时发送 |
| Write | POLLOUT | socket 的发送缓存有足够的空间写入数据 |
| Read/Write | POLLIN \| POLLOUT | 向外的 `connect` 完成 (可以读取或者发送数据) |
| Read/Write | POLLERR | 出现异步错误 |
| Read/Write | POLLHUP | 对端单向关闭链接 |
| Exception | POLLPRI | 紧急数据到达. SIGURG 信号同时发送 |

另一个 `poll` 和 `select` 是通过系统内核向应用程序发送 `SIGIO` 信号通知. 这种情况下, 必须使用 `fcntl` 对 socket 文件描述符设置 `O_ASYNC` 标志并且通过 `sigaction` 安装 `SIGIO` 信号的处理函数.

## Socket 地址的数据结构

每个 socket 域都有自己特有的地址数据结构. 每个这种数据结构都会以 `sa_family_t` 类型(int)的 `family` 字段来标识地址数据结构. 这种结构允许很多的系统调用(例如: `connect`, `bind`, `accept`, `getsockname`, `getpeername`)基本上可以处理所有的 socket 域, 然后通过不同的地址结构类型来区分.

系统抽象出通用的 `struct sockaddr` 数据结构来处理所有 socket 域特有的地址数据结构.

另外, sockets API 也提供了 `struct sockaddr_storage`. 这个结构可以容纳所有类型的 socket 地址结构；它容量足够并且正确对齐. (事实上, 它足够容纳 IPv6 地址). 此结构通过下面的字段来区分不同的 socket 地址类型:

```c
sa_family_t ss_family;
```
`sockaddr_storage` 对想要统一处理 socket 地址的程序非常有用. (例如: 程序需要同时处理 IPv4 和 IPv6).

## Socket 选项
以下列举的所有选项都可以使用 `setsockopt` 和 `getsockopt` 通过设置 `level` 参数为 `SOL_SOCKET` 来设置或者获取. 除非特殊说明, `optval` 参数是 int 指针.

### #函数原型

```c
#include<sys/types.h>
#include<sys/socket.h>

int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
```

### #SO_ACCEPTCONN

返回信息指示当前的 socket 是否通过 `listen` 设置了接受链接. 返回 0 表示不是, 1 表示是. 此选项是`只读`的.

### #SO_ATTACH_FILTER (Linux 2.2 - ), SO_ATTACH_BPF (Linux 3.19 - )

对 socket 绑定一个 经典 BPF (`SO_ATTACH_FILTER`) 或者拓展的 BPF (`SO_ATTACH_BPF`) 用来过滤接收的数据包. 当过滤器程序返回 0 的时候直接丢弃数据包. 当过滤器返回非零值但是小于数据包中数据的长度, 那么数据包会按照返回的长度截取数据.当返回值大于或者等于数据包中数据的长度, 数据可以不做处理.

`SO_ATTACH_FILTER` 选项的参数值定义在 `<linux/filter.h>` 文件中, 是一个 `sock_fprog` 结构.

```c
struct sock_fprog {
    unsigned short     len;
    struct sock_filter *filter;
};
```

`SO_ATTACK_BPF` 选项的参数值是 `bpf(2)` 系统调用返回的文件描述符, 并且必须关联 `BPF_PROG_TYPE_SOCKET_FILTER` 类型的程序.

此配置项在同一个 socket 上面可能会设置多次, 每次都会覆盖掉原来的配置. 经典 BPF 和拓展的 BPF 可能会绑定到同一个 socket, 但是每次都会替换掉原来的 BPF, 同一时刻在同一个 socket 上面只会有一个 BPF.

查看 `Documentation/networking/filter.txt` 文件了解更多.

```
BPF (Berkeley Packet Filter) 伯克利包过滤器.
```

### #SO_ATTACH_REUSEPORT_CBPF, SO_ATTACH_REUSEPORT_EBPF

和 `SO_REUSEPORT` 选项一起使用, 这些选项允许用户设置一个经典 BPF (通过 `SO_ATTACH_REUSEPORT_CBPF`) 或者拓展 BPF (通过 `SO_ATTACH_REUSEPORT_EBPF`) 程序来定义数据包怎样分派到重用端口组的 socket 中. (`重用端口组 socket`: socket 开启了 `SO_REUSEPORT` 选项并且使用相同的本地地址接收数据包)

BPF 程序必须返回 0 到 N-1 之间的数值来表示哪个 socket 来接收数据包(`N` 表示重用端口组中的 socket 数量). 如果返回了非法的数值, 就会通过原始的 `SO_REUSEPORT` 来选择 socket 接收数据.

重用端口组中的 socket 是按照加入的顺序来排序(即: UDP socket 按照调用 `bind` 的顺序； TCP socket 按照调用 `listen` 的顺序). 新加入的 socket 会继承已经绑定的 BPF. 当组中某个 socket 被移除( socket 调用了 `close` 方法), 那么组中的最后一个 socket 会被移动到被移除 socket 的位置.

可以在任何时候对组中的任意 socket 设置 BPF, 新设置的 BPF 会替换掉之前的, 并且 BPF 会作用与组中的所有 socket.

`SO_REUSEPORT_CBPF` 和 `SO_ATTACH_FILTER` 使用相同的参数类型, `SO_REUSEPORT_EBPF` 和 `SO_ATTACH_BPF` 使用相同的参数类型.

UDP 从 Linux 4.5 开始支持; TCP 从 Linux 4.6 开始支持.

### #SO_BINDTODEVICE

将 socket 和指定的设备绑定, 例如: `eth0`. 当传入的 `name` 是空字符串或者 `len` 长度为0, 绑定失败. 传入的选项是一个可变长度空终结的接口名称字符串, 最大长度是 `IFNAMSIZ` (16字节). 当 socket 绑定到接口设备上面之后, 只有这个接口设备收到的数据包才会传递给 socket 进行处理. 注意: 此选项之对某些 socket 有用, 特别是 `AF_INET` 类型. 此选项不支持 `Packet socket`(L2层的 socket), 但是可以使用 `bind(2)` 来绑定.

Linux 3.8 之前, 此配置可以设置但是使用 `getsockopt` 获取不到. `optlen` 参数包含的缓存尺寸应该能够接收设备名称, 建议为 `IFNAMSIZ` 个字节. 真实的设备名称通过 `optlen` 返回.

### #SO_BROADCAST

设置或者获取广播标志. 开启时, 数据报文 socket 允许发送数据包到广播地址. 此配置不会影响面向流的 socket.

### #SO_BSDCOMPAT

开启 BSD 完全兼容. 在 Linux 2.0 和 2.2 版本中被 UDP 协议模块使用. 开启时, UDP socket 接收到的 ICMP 错误不会传递到用户程序. 在后来的 Linux 内核, 此选型的支持被逐步淘汰: Linux 2.4 默默的忽略, Linux 2.6 中如果用户使用了此选项会生成内核警告(`prinkt()`). 

### #SO_DEBUG

开启 socket 调试. 只有在程序 `CAP_NET_ADMIN` 兼容或者有效的用户ID 0 时候才能使用.

### #SO_DETACH_FILTER(Linux 2.2 - ), SO_DETACH_BPF( Linux 3.19 - )

这两个选项意思相同, 可能会被用做移除已经通过 `SO_ATTACH_FILTER` 或者 `SO_ATTACH_BPF` 绑定的经典或者拓展 BPF. 此选项的 `value` 参数会被忽略.

### #SO_DOMAIN (Linux 2.6.32)

获取 socket 域并返回 integer, 例如返回 `AF_INET6`. 此选项是只读的.

### #SO_ERROR

获取并清空待定的 socket 错误. 此选项是只读的. 期望 integer.

### #SO_DONTROUTE

不通过网关发送数据, 只发送给直连主机. 可以在 `send(2)` 中使用 `MSG_DONTROUTE` 标志来达到同样的效果. 期望整数形式的布尔值.

### #SO_INCOMING_CPU

设置或者获取 socket 和 CPU 的亲和性. 期望 integer 标志.

```c
int cpu = 1;
setsockopt(fd, SOL_SOCKET, SO_INCOMING_CPU, &spu, sizeof(cpu));
```
同一数据流中的所有数据包都由一个和指定 CPU 关联的 RX 队列(`接收队列`)接收, 通常的做法就是每个 RX 队列使用一个监听处理程序, 所有接收的数据包都由和 CPU 绑定的监听处理程序来处理. 这也提供了最佳的 NUMA(Non-Uniform Memory Access, 非一致内存访问)并且保持 CPU 缓存不过期.

### #SO_KEEPALIVE

在面向链接的 socket 上面开启发送 keep-alive(保活) 信息的功能.  期望整数形式的布尔值.

### #SO_LINGER

设置或者获取 `SO_LINGER` 选项. 参数是 `linger` 结构:

```c
struct linger {
    int l_onoff;   /* linger active */
    int l_linger;  /* how many seconds to linger for */
};
```

当开启此选项, 调用 `close(2)` 或者 `shutdown(2)` 的时候会等到当前 socket 发送缓存中的所有数据全部发送成功或者达到了 linger 的超时时间, 才会返回. 否则, 调用会直接返回并且在后台关闭链接. 如果 socket 是因为 `exit(2)` 而被动关闭, 通常都在后台进行.

### #SO_LOCK_FILTER

当设置了此选项的时候, 会阻止更改 socket 关联的过滤器. 这些过滤器是通过 `SO_ATTACH_FILTER`,`SO_ATTACH_BPF`, `SO_ATTACH_REUSEPORT_CBPF` 和 `SO_ATTACH_REUSEPORT_EBPF` 这些选项设置的.

通常的使用方式是, 特权进程创建一个 `Raw Socket(需要 CAP_NET_RAW 功能)`, 应用一个过滤器, 设置 `SO_LOCK_FILTER` 选项, 然后放弃特权或者通过 `UNIX Domain Socket` 将当前 socket 的文件描述符传递给另外一个非特权的进程.

一旦开启了 `SO_LOCK_FILTER` 选项, 企图修改或者删除 socket 已绑定的过滤器, 或者取消 `SO_LOCK_FILTER` 选项都会失败并返回 `EPERM` 错误.

### #SO_MARK (Linux 2.6.25)

对当前 socket 发送的所有数据包设置标记(和 `netfilter MARK` 很像, 但是此选项是针对 scoket的). 改变标记可以用于实现基于标记的路由, 不用使用 netfilter 或者数据包过滤. 设置此选项需要获取 `CAP_NET_ADMIN` 权限.

### #SO_OOBINLINE

如果开启此选项, `out-of-band(带外数据, 紧急数据)` 直接放到 socket 的接收数据流中. 否则,
`out-of-band(带外数据, 紧急数据)` 只有在接收数据时设置了 `MSG_OOB` 标志才会传递.

### #SO_PASSCRED

是否开启接收 `SCM_CREDENTIALS` 控制消息. 查看 `unix(7)` 了解更多.

### #SO_PASSSEC

是否开启接收 `SCM_SECURITY` 控制消息.  查看 `unix(7)` 了解更多.

### #SO_PEEK_OFF (Linux 3.4 - )
```
MSG_PEEK 返回数据但是不会删除数据, recv() 不带 MSG_PEEK 返回数据并且删除已返回的数据
```

此选项目前仅适用于 `unix(7)` socket, 作用是设置系统调用 `recv(2)` 的 `peek offset` 偏移量, 和 `MSG_PEEK` 标志配合使用.

当此选项设置为负值(新创建的 socket 设置为 -1 )的时候, 传统行为是:
带有 `MSG_PEEK` 标志的 `recv` 调用会从队列头开始 `peek` 数据.

当此选项设置非负值( >= 0 )时, 下一次 `peek` 接收队列的数据是从此选项指定的字节数开始. 同时, `peek offset` 会基于之前的已经 `peek` 的偏移量增加, 所以接下来的 `peek` 会返回队列中偏移量随后的数据.

如果接收队列前端的数据被不带 `MSG_PEEK` 的 `recv(2)` 调用删除, `peek offset` 会随之减少删除的字节数. 换句话说, 不带 `MSG_PEEK` 的 `recv(2)` 会导致 `peek offset` 重新调整来维护正确的位置, 所以接下来的 `peek` 调用会取回那些已经取回过的但是没有删除的数据.


对于数据报 socket (UDP socket) 来说, 如果 `peek offset` 指向一个数据包的中点, 返回的数据带着 `MSG_TRUNC` 标记.

下面的代码阐述了 `SO_PEEK_OFF` 的用法. 假设一个数据流 socket 接收到以下数据:
```
aabbccddeef
```
接下来的 `recv(2)` 调用的影响都体现在代码注释中:
 
```c
int ov = 4; //设置 peek offset 为 4, 表示 peek 从 4 开始
setsockopt(fd, SOL_SOCKET, SO_PEEK_OFF, &ov, sizeof(ov));

recv(fd, buf, 2, MSG_PEEK);  // Peek "cc"; offset set to 6
recv(fd, buf, 2, MSG_PEEK);  // Peek "dd"; offset set to 8
recv(fd, buf, 2, 0);         // Reads "aa"; offset set to 6
recv(fd, buf, 2, MSG_PEEK);  // Peeks "ee"; offset set to 8

```

### #SO_PEERCRED

返回链接到 socket 的 peer 进程的认证信息. 查看 `unix(7)` 了解更多.

### #SO_PRIORITY

为当前 socket 准备发送的所有数据包设置 `protocol-defined(协议定义)` 的优先级. Linux
使用此值来对网络队列进行排序: 拥有高优先级的数据包会根据选中的网络设备队列纪录优先处理. 设置 0-6 之外的优先级需要程序获取到 `CAP_NET_ADMIN` 权限.

### #SO_PROTOCOL (Linux 2.6.32 - )

获取当前 socket 的协议类型并以 Integer 形式返回, 例如: `IPPROTO_SCTP`. 查看 `socket` 了解更多. 此选项是只读的.

### #SO_RCVBUF

获取或者设置 socket 接收缓存的最大字节数. 当使用 `setsockopt(2)` 来设置的时候, `内核会将此值乘以 2 `(为了允许保存内核对数据的前置标记信息), 使用 `getsockopt(2)` 获取的时候也会返回 `此值乘以 2 ` 的结果. 默认值设置在 `/proc/sys/net/core/rmem_default` 文件, 最大值设置在 `/proc/sys/net/core/rmem_max` 文件. 最小值 (`乘以2`) 是 256.

### #SO_RCVBUFFORCE (Linux 2.6.14 - )

特权进程 (获取了 `CAP_NET_ADMIN` 权限) 使用此  socket 选项和设置 `SO_RCVBUF` 选项效果一样, `但是 rmem_max 可以被重写`.

### #SO_RCVLOWAT, SO_SNDLOWAT

socket 缓存中数据的字节数达到此值时, 才会发送给底层协议栈 (`SO_SNDLOWAT`) 或者上层等待接收的用户程序(`SO_RCVLOWAT`). 这两个选项初始值都是 `1`. Linux 上 `SO_SNDLOWAT` 选项不可改变(`setsockopt(2)`会失败并返回 `ENOPROTOOPT` 错误). Linux 2.4 之后才能设置 `SO_RCVLOWAT` 选项.

Linux 2.6.28 之前, 使用 `select(2)`, `poll(2)` 和 `epoll(2)` 不会遵循 `SO_RCVLOWAT`设置, 只要 socket 中有可读数据就会直接返回. 接下来从 socket 读取的时候会一直阻塞, 直到 socket 接收缓存达到 `SO_RCVLOWAT` 个字节. Linux 2.6.28 开始, 只有最少 `SO_RCVLOWAT` 字节可读的时候, `select(2)`, `poll(2)` 和 `epoll(2)` 才会返回.



