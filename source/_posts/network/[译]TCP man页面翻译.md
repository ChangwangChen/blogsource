---
title: [译]TCP man页面翻译
date: 2020-07-27 13:50:45
categories: network
tags: [Network]
---

# 说明
本文翻译自 [Tcp Man7](https://man7.org/linux/man-pages/man7/tcp.7.html), 并且只是翻译了 Description 部分内容。

# 翻译

这是对 `RFC793`、 `RFC1122`和`RFC2001`及其`NewReno`和`SACK`拓展规定的 TCP 协议的一个实现。它提供了一个建立在 IPv4 和 IPv6 基础上面的`可靠的`、`面向流的`、`全双工`的两个`socket`之间的链接。TCP 保证了数据到达的顺序和对丢失包的重传。它通过对每个数据包生成和检验`checksum`来发现传输错误。 TCP不保留记录的边界。

一个新创建的 TCP socket 没有远端和本地的地址， 并且也没有完全指定。通常使用 `connect` 和另外一个 TCP socket 建立链接作为向外的 TCP链接。为了接收新传入的 TCP 链接， 首先需要使用 `bind` 来绑定 socket 和本地的`地址`以及`端口号`， 然后使用 `listen` 将 socket放到监听状态。在这之后，socket 每次都通过 `accept` 来接受新的连接。 socket 只有在 `accept` 或者 `connect` 成功之后才被完全指定， 并且能够传输数据。

Linux 支持 `RFC1323` TCP 高性能拓展。其中包括 `PAWS(防止重复序列号)`、`Window Scaling（窗口缩放）`和 `时间戳`。`Window Scaling（窗口缩放）`允许使用较大的 TCP window （>64KB 1<<16）来支持高延迟或者高带宽的链接链路。要使用它们，发送和接收的缓冲区也必须同时增大。 发送和接收缓冲区可以通过修改 `/proc/sys/net/ipv4/tcp_wmem` 和 `/proc/sys/net/ipv4/tcp_rmem` 文件来全局调整， 也可以使用 `setsockopt`方法通过设置 `SO_SNDBUF` 和 `SO_RCVBUF`选项对单独的 socket 进行调整。

通过设置 `SO_SNDBUF` 和 `SO_RCVBUF` 选项来修改 socket 缓冲区尺寸的最大大小分别在 `/proc/sys/net/core/rmem_max` 和 `/proc/sys/net/core/wmem_max` 文件中限制。需要注意的是， TCP 实际会分配`setsockopt`设置缓冲尺寸大小的 **2倍**, 所以通过 `getsockopt` 获取的缓冲区尺寸会和当初设置的不一致。TCP使用多出来的缓冲区来实现管理和保存内核的数据结构，并且在 `/proc` 文件中反映的也是和 TCP 真实窗口大小相比的较大值。对于每个 socket 链接来说， 缓冲区尺寸必须在调用 `listen` 或者 `connect` 之前进行设置才能发挥作用。查看 `scoket` 了解更多。

TCP 支持发送 `紧急数据(Urgent Data)`。 `紧急数据`用来提醒接受方有些重要的数据在当前的数据流中， 并且需要尽快处理这些数据。可以通过在 `send` 方法中添加 `MSG_OOB` 来发送 `紧急数据`。进程或者进程组可以通过调用 `ioctl` 方法并设置 `SIOCSPGRP` 或者 `FIOSETOWN` （或者使用 POSIX.1指定的 `fcntl` 并设置 `F_SETOWN`）来监听 socket。当接收到紧急数据时， 内核会发送 `SIGURG` 信号给这些进程或者进程组。当 socket 开启了 `SO_OOBINLINE` 选项， 紧急数据会被放在普通的数据流中（下面会使用 `SIOCATMARK` ioctl 选项来测试）， 否则紧急数据只能通过调用 `recv` 或者 `recvmsg` 方法并设置 `MSG_OOB` 标志来获取。

当出现了 `带外数据(out-of-band data)`， `select` 会返回出当前的文件描述符出现了一个特殊选项， 而 `poll` 会返回一个 `POLLPRI` 事件。

Linux 2.4 介绍了一系列的增加 TCP 吞吐量和伸缩性的改变， 以及一些高级功能。其中包括支持 `sendfile` 的零拷贝， 显式拥塞通知， TIME_WAIT 状态 socket 的新的管理， socket Keep-Alive选项和支持 复制 SCK 拓展。

## 地址格式

TCP 建立在 IP 协议之上， 所以 IP 的地址格式在 TCP 也适用`（IP 接口地址和 16位 的端口号）`。 TCP 支持 `点对点(point-to-point)` 的通信， 不支持 广播 和 多播。

## /proc 接口
系统级的 TCP 参数设置可以通过修改 `/proc/sys/net/ipv4/` 目录下的文件实现。另外， 大多数的 IP `/proc` 接口对 TCP 也同样适用。Boolean 类型的参数使用 integer 来表示， 非零值表示该选项开启， 零值表示关闭。

### #tcp_abc(Integer; 默认 0; Linux 2.6.15 - Linux 3.8)
控制 `适当字节数(Appropriate Byte Count)`, `RFC 3465`中定义。
`ABC`是在部分确认过程中增加 `拥塞窗口(cwnd)` 更加缓慢的一种方式。可能的值如下：

- 0 每收到 ack， cwnd 加 1 （没有 ABC）
- 1 每收到全尺寸段的 ack， cwnd 加 1
- 2 当接收到 ack 是对延迟确认的两个段的响应的时候， 允许将 cwnd 加 2

### #tcp_abort_on_overflow (Boolean; 默认: Disabled; Linux 2.4 - )

当链接过多导致服务器无法处理(accept)的时候， 允许服务器返回 RST 让客户端重新链接。这意味着当服务器拥塞导致全链接队列（accept队列）溢出的时候， 链接将会自动恢复。只有在你知道服务器真的无法更快的处理链接的时候开启这个选项。开启这个选项会导致客户端重新发送 SYN 请求链接。

- 0 （Disable）： 当 accept 队列溢出的时候， 服务器会直接丢弃客户端链接
- 1 （Enable） ： 当 accept 队列溢出的时候， 服务器会发送 RST 报文， 客户端收到之后会重新开启三次握手 

### #tcp_adv_win_scale (Integer; 默认： 2; Linux 2.4 - )

当 tcp_adv_win_scale 大于 0 的时候， 计算缓存开销为 bytes/2^tcp_adv_win_scale; 小于等于 0 的时候， 为 bytes - bytes/2^(-tcp_adv_win_scale)。

socket 的接收缓存空间是和系统缓存共享的。TCP 只维护接收缓存空间的一部分作为 TCP 的窗口大小， 这也是给对方的发送窗口大小。剩余空间用作操作系统缓冲区， 用来隔离系统对 socket 的调度和延迟。tcp_adv_win_scale 等于 2 暗示会拿出 1/4 的空间作为系统的缓冲区。

### #tcp_allowed_congestion_control (String; 默认: see text；Linux2.4.20 - )

展示或者设置对非特权进程可用的拥塞控制算法（详情查看 socket 中 `TCP_CONGESTION` 配置项的描述）。该内容以`空格`分隔， 并使用`换行符号`结束。该算法只能在 `tcp_available_congestion_control` 规定的选项中选择。该配置的默认值是 "reno" 加上 `tcp_congestion_control` 配置项的默认值。

### #tcp_autocorking （Boolean； 默认：1； Linux 3.14 - ）

当这个选项开启， 内核会试着尽可能多的合并小的写入数据（从持续的 `write` 和 `sendmsg` 系统调用中）来减少数据包的发送数量（`使用 Nagle 算法`）。当至少有一个预先数据包在 `Qdisc Queue(排队规则队列)` 或者 `设备传输队列` 中等待时， 合并完成。程序仍然可以使用 `TCP_CORK` 选项保持 socket 的最佳行为状态。

### #tcp_available_congestion_control（String; 只读选项； Linux 2.4.20 - ）

展示系统已经注册的拥塞控制算法。该内容以`空格`分隔， 并使用`换行符号`结束。`tcp_allowed_congestion_control` 配置项只能在这里展示的内容中选择。更多的拥塞算法以模块的方式提供， 但是没有加载。

### #tcp_app_win(Integer; 默认： 31； Linux 2.4 - )

此配置项定义了 TCP 窗口中会保留多少字节给应用程序缓存使用。计算公式为：

```
window / 2 ^ tcp_app_win
```
0 表示不保留。

### #tcp_base_mss(Integer; 默认： 512; Linux 2.6.17 - )

`search_low` 是 TCP 打包层路径 MTU 发现（`MTU探测`）的初始值。如果开启了 MTU 探测， tcp_base_mss 是初始的 MSS 大小。

### #tcp_bic （Boolean； 默认： 0； Linux 2.4.27 / 2.6.6 - 2.6.13）

开启 BIC TCP 拥塞控制算法。`BIC-TCP`算法发送方特有的， 它的目的是为了在大窗口环境提供可拓展性和有限TCP友好性的同时， 保证线性 RTT 公平性。它结合了两种策略： 增量增长和二分查找增长。当拥塞窗口较大时， 使用增量增长保证了线性 RTT 的公平性和拓展性；拥塞窗口较小时， 使用二分查找增长， 提供了 TCP 的友好性。

### #tcp_bic_low_window （Integer；默认：14； Linux 2.4.27 / 2.6.6 - 2.6.13）

当时使用 BIC TCP 算法开始调整拥塞窗口大小时候， `tcp_bic_low_window` 是窗口大小的阈值。当拥塞窗口小于这个阈值， BIC TCP 的算法和 TCP Reno 一样。

### #tcp_congestion_control （String； 默认： see text； Linux 2.4.13 - ）

设置 TCP 新链接使用的默认拥塞控制算法。`Reno`算法总是能够使用的， 其他的算法是否能够使用由内核配置。此配置项是内核拥塞控制算法的一部分。

### #tcp_dma_copybreak （Integer； 默认： 4096； Linux 2.6.24 - ）

在 Linux 内核开启了 `CONFIG_NET_DMA` 选项的时候， 此配置项的值设置了由 DMA 引擎直接从 socket 缓存中复制数据到用户空间的字节数`下限`。

### #tcp_dsack （Boolean； 默认： 1； Linux 2.4 - ）

是否开启 RFC 2883 TCP DSACK 支持。

### #tcp_ecn （Integer； 默认： 查看下文； Linux 2.4 - ）

开启 RFC 3168 显式拥塞通知。
取值如下：
- 0  关闭 ECN。不准发起和接受ECN。Linux 2.6.30 开始的默认值。
- 1  接收和发起链接上面都开启 ECN。
- 2  只在接收的链接上面开启 ECN。Linux 2.6.31之后的默认值

当开启的时候， 链路上旧的或者行为怪异的中间包会影响对一些终端的联通性， 从而导致链路关闭。因此， 为了促进和鼓励部署选项 1， `tcp_ech_fallback`就出现了。

### #tcp_ecn_fallback （Boolean；默认：1；Linux 4.1 - ）

参照 RFC 3168, 6.1.1.1 fallback 小节。当开启的时候， 如果设置了 ECN 标志的 SYN 报文超时时间超过了普通 SYN 重传超时时间， TCP 会重传清除了 `CWR` 和 `ECE`标志的 SYN 报文。

### #tcp_fack （Boolean；默认： 1； Linux 2.2 - ）

启用 TCP 转发应答支持。

### #tcp_fin_timeout （Integer； 默认： 60； Linux 2.2 - ）

TCP 在 `FIN_WAIT_2` 状态下强制关闭链接的延迟时间， 单位是： `秒`。这个配置项严重的违反了 TCP 规范， 但是也防止了 `Denial-of-Service Attacks(DoS Acctack： 拒绝服务攻击)`。在 Linux 2.2 版本， 默认值是 180（秒）。

### #tcp_frto （Integer； 默认： 查看下文； Linux 2.4.21/2.6 - ）

```
F-RTO 虚假重传超时探测
```
开启 F-RTO， TCP 重传超时（RTOs）的高级恢复算法。这个算法尤其对无线环境有利， 因为无线环境重传超时通常是因为无线电干扰而不是中间路由器拥塞。更多请查看 RFC 4138。

取值如下：
- 0 关闭。Linux 2.6.33 及之前的默认值。
- 1 基础版本的 F-RTO 算法开启。
- 2 当流使用了 `SACK`的时候， 使用 `高级SACK` F-RTO 算法。当使用了 `SACK`， 但是 `F-RTO` 和开启了 `SACK` 的流交互不好的时候， 也可以使用基本版本的算法。 Linux 2.6.24 开始默认是此值。

Linux 2.6.22 之前， 次配置项是 Boolean， 并且只支持上面的 0 和 1。

### #tcp_frto_response （Integer；默认：0； Linux 2.6.22 - ）

当 `F-RTO` 探测到一个 TCP 重传超时是虚假的时候（例如： 调高 TCP 重传超时时间会消失）， TCP 有以下几种选项：

- 0 基数减半；平滑且保守的响应策略。在一个 RTT 时候， 拥塞窗口（`cwnd`）尺寸和慢启动阈值(`ssthresh`)都会减半。
- 1 非常保守响应。此选项虽然可用但是`不推荐使用`，因为和其他 TCP 的交互很差。直接将拥塞窗口（`cwnd`）尺寸和慢启动阈值(`ssthresh`)减半。
- 2 激进响应。取消拥塞控制措施被认为是不必要的（忽略了大量重传的可能性， 这时候 TCP 更加谨慎）。拥塞窗口（`cwnd`）尺寸和慢启动阈值(`ssthresh`)会被设置成出现超时之前的值。

### #tcp_keepalive_intvl （Integer； 默认： 75； Linux 2.4 - ）

TCP Keep-Alive（保活）探测的时间间隔`秒`数。

### #tcp_keepalive_probes （Integer； 默认： 9； Linux 2.2 - ）

在对端连续对 `tcp_keepalive_probs` 个 `保活探测包` 无响应之后， TCP 决定放弃探测并关闭当前链接。

### #tcp_keepalive_time （Integer； 默认： 7200； Linux 2.2 - ）

TCP 在经过 `tcp_keepalive_time 秒` 的空闲（无数据交流）之后， 决定发起 `保活探测`。这个只有在 socket 设置了 `SO_KEEPALIVE` 标志的时候才会执行。默认时间是 `7200s （2 hours）`。一个空闲在链接大约需要在等 `11 分钟（再连续发送 9 个 间隔默认为 75 秒的探测包）` 之后， 才会被真正的关闭。

注意： 底层链接的追踪机制和程序的超时时间可能更短。

### #tcp_low_latency （Boolean；默认：0；Linux 2.4.21/2.6 - ； Linux 4.14 及以后过期）

如果开启， TCP 协议栈将会优先选择 `低延迟`， 反之则选择 `高吞吐`。程序需要改变默认设置的例子就是 `Beowulf compute cluster`。从 Linux 4.14 开始， 此配置文件依然存在但是 TCP 直接将其忽略。

### #tcp_max_orphans （Integer；默认： 查看下文； Linux 2.4 - ）

系统允许最大的`孤儿socket（没有任何用户文件处理与之绑定）` 数目。当数量达到之后，这些链接会被重置并打印警告（`Out of memory`）。此配置项主要是为了阻止简单的 DoS 攻击。`不建议调低此值`。需要根据网络情况来调高此配置，但是要记住每个孤儿socket会占用 `～64KB` 的不可置换内存空间。此配置项的初始值和内核参数 `NR_FILE` 一致。该内核配置项的初始值就根据系统的内存大小调整。

### #tcp_max_syn_backlog（Integer; 默认：查看下文; Linux 2.2 - ）

`半链接队列（没有收到三次握手中最后一个 ACK 的链接）` 最大数目。达到最大数（半链接队列满了）之后， 内核开始丢弃客户端的 SYN 请求。当系统内存充足或者大于（128 MB）的时候， 此配置项的默认值会从 256 调整到 1024, 系统内存小于 （32 MB） 则会降低到 128。

在 Linux 2.6.20 之前， 如果想要将此配置项调整到大于 1024, 则需要在内核文件 `include/net/tcp.h` 中将表示 `SYNACK Hash Table` 大小的宏 `TCP_SYNQ_HSIZE` 相应的调整为 
```
TCP_SYNQ_HSIZE * 16 <= tcp_max_syn_backlog
```
, 并且需要重新编译内核。在 Linux 2.6.20 中， 内核将 `TCP_SYNQ_HSIZE` 从固定值调整为动态获取。

### #tcp_max_tw_buckets （Integer； 默认： 查看下文； Linux 2.4 - ）

系统允许处于 `TIME_WAIT` 状态 socket 的最大数目。此配置项的目的是为了阻止简单的 DoS 攻击。默认值是 `NR_FILE * 2`, 根据系统的内存大小自动调整。当达到最大数目时， socket 会直接关闭并打印警告。

### #tcp_moderate_rcvbuf （Boolean；默认：1； Linux 2.4.17/2.6.7 - ）

如果开启， TCP 会自定调整接收缓存的大小（`不会超过 tcp_rmem`） 来匹配链路吞吐量。

### #tcp_mem （Linux 2.4 - ）

此配置项是 3 个数组成的向量， 分别表示： [`low`, `pressure`, `high`]。这些参数规范的单位是系统内存页， TCP 使用他们来检测内存使用。默认值会在启动的时候根据系统内存大小来计算（TCP 只会使用 `low` 配置项， 32位系统大概是 900 MB， 64位系统没有限制）。

- low     
  TCP 占用的内存页数小于此值的时候，不会在调整。
- pressure  
  当 TCP 占用的内存页数达到此值的时候，TCP 会减慢内存的使用。一旦使用内存页数小于 `low` 的时候， 退出压力状态。
- high
  TCP 全局能够占用的最大内存页数。所有内核对 TCP 内存的限制都会被此值重写。

### #tcp_mtu_probing（Integer；默认： 0；Linux 2.6.17 - ）

此配置控制 TCP 在 `Packetization-Layer` 路径 MTU 的发现。可能的取值如下：

- 0 关闭
- 1 默认关闭， 当检测到 ICMP 黑洞的时候开启
- 2 开启， 使用初始 MSS 作为 `tcp_base_mss`

### #tcp_no_metrics_save （Boolean；默认： 0；Linux 2.6.6 - ）

TCP 默认会在链接关闭的时候保存大量的链接指标进路由缓存中， 方便在接下来的短时间内再次建立链接时进行参数初始化。这样做通常会提高 TCP 的整体性能， 但是有时也会造成性能衰减。当开启 `tcp_no_metrics_save` 的时候， TCP 不再链接关闭的时候保存这些信息。

### #tcp_orphan_retries （Integer；默认： 8； Linux 2.4 - ）

对 `孤儿socket` 的对方是否关闭进行探测的最大次数。（`注意这里设置成 0， 并不意味这一定是 0`）

### #tcp_reordering （Integer；默认： 3；Linux 2.4 - ）

在 TCP 没有假设丢包和即将进入慢启动时， 数据包的在 TCP 流中能够重新排序的最大次数。`一般不建议修改此值`。这是在一个链接对包的重新排序过程中， 最大程度减少数据包回退操作和重传的检测指标。

### #tcp_retrans_collapse （Boolean；默认： 1；Linux 2.2 - ）

TCP 重传过程中是否发送全尺寸的数据包。

### #tcp_retries1 （Integer；默认： 3；Linux 2.2 - ）

TCP 不需要 IP 层参与的情况下对数据包重传的尝试次数。当 TCP 重传次数达到时， 首先会让 IP 层更新路由， 然后在进行下次重传。默认值 3 是 TCP 规范的最小值。

### #tcp_retries2 （Integer；默认：15； Linux 2.2 - ）

TCP 在建立状态链接上对数据包放弃传输之前重传的最大次数。默认值是 15, 根据重传超时时间不同可能会持续 13 到 30 分钟。RFC 1122 规范中最小限制 100 秒通常被认为比较短。

### #tcp_rfc1337 （Boolean；默认： 0；Linux 2.7 - ）

TCP 是否遵循 RFC 1337。当关闭时， 处于 TIME_WAIT 状态的链接收到 RST 包直接关闭链接而不用等到 TIME_WAIT 结束。

### #tcp_rmem （Linux 2.4 - ）

此配置项是包含了 3 个值的向量： [`min`, `default`, `max`]。TCP 使用此配置项来调节接收缓存的尺寸。TCP 会根据以下规则对接收缓存进行动态调整：

- min  
  每个 TCP 链接接收缓存使用的最小尺寸。此值的默认值是 `系统内存页（Linux 2.4 版本中默认值是 4KB， 低内存系统是 PAGE_SIZE）`。此值确保在 TCP 内存 `pressure` 状态下， 在申请小于 `min` 大小的内存时都成功。此配置项不会界定通过 socket 选项 `SO_RCVBUF` 设置的接收缓存尺寸。
- default
  TCP socket 的默认接收缓存尺寸。配置此值会重写 TCP 全局 `net.core.rmem_default` 配置的初始缓存尺寸。默认大小是 87380 `字节`（Linux 2.4 版本， 低内存系统是 43689 `字节`）。想要扩大接收缓存的尺寸可以直接调高此值（`会影响到所有 TCP socket`）。想要增大 TCP 窗口尺寸， `net.ipv4.tcp_window_scaling` 必须开启（默认就是开启）。
- max
  TCP socket 接收缓存的最大尺寸。此值不会重写 `net.core.rmem_max` 的配置， 也不会限制 socket `SO_RCVBUF` 选项。通过以下方程式计算默认值：
  ```
  max(87380, min(4 MB, tcp_mem[1]*PAGE_SIZE/128))

  (Linux 2.4，默认值是 87380 * 2 字节， 低内存系统是 87380 字节)
  ```

### #tcp_sack （Boolean；默认： 1；Linux 2.2 - ）

是否开启 RFC 2018 TCP 选择回复。

```
SACK：
TCP 接收方在接收异常（例如：数据流产生空洞， 数据包乱序）的时候，在 ACK 报文中主动返回导致乱序的数据段信息。

D-SACK：

```










