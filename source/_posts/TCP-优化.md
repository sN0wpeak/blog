---
title: TCP-优化
date: 2023-01-17 13:09:15
tags: tcp
---

# 简介

tcp面向字节流的可靠链接协议.

## TCP 3次握手.

握手主要是协商option和

相关优化

``` shell
sysctl -w net.ipv4.tcp_syn_retries=2
sysctl -w net.ipv4.tcp_synack_retries=2
sysctl -w net.core.netdev_max_backlog=32768
sysctl -w net.ipv4.tcp_max_syn_backlog=32768
sysctl -w net.ipv4.tcp_fastopen=3
```

## TCP传输

tcp协议会从应用发来的任意长度的报文段，按照MSS拆分为报文段进行发送.
> MSS max segment size. TCP数据的大小，不包含tcp头. 握手阶段协商MSS

### 重传

RTT(Round Trip Time)  是传送到接受的时间, RTT是会波动的.  
保证segment段一定能发送到目标端的方式是通过重传和确认.
因为TCP报文是有序的，所以必须保证保证上一条消息确认后才能处理下一条消息，如果超过一段时间（RTO Retransmission
TimeOut）没有收到ACK，会重传. 如果每次发送都要等待一次ack的话(PAR)，效率太低了。所以出现了滑动窗口，可以并发进行发送和确认.

![img_1.png](/images/img_1.png)
![img.png](/images/img.png)

由于RTT会有波动（受中间点节点变化等等原因），所以需要设置一个合理的RTO值才能得到更好的网络性能，往往比RTT大一点就可以.
那怎么动态设置呢？ 通过RFC6298提出的追踪RTT方差来实现.

TCP使用SEQ和ACK来确认保证字节流是有序的和确认字节，SEQ最大是2^32次方. 如果序列号用完了会从0开始继续.
如果出现回绕和重传的问题，使用PAWS(timestamp)防止回绕.

### 滑动窗口

滑动窗口分为发送窗口和接受窗口.

发送窗口:
![img_2.png](/images/img_4.png)
发送窗口分为4部分
已经发送并确认. 发送未确认. 待发送可以发送. 超出窗口大小等待可用空间. 
接受爽口: 
接受窗口分为3部分
已经接受. 接受准备处理. 未接受 

发送窗口取决于当前的接受窗口. 

窗口受操作系统缓冲区的约束，也会随之收缩的. 比如当前500个链接，缓存还够. 忽然到了1000个就需要适当较少窗口大小了.而且操作系统的缓冲区除了TCP窗口在用，还会用于应用缓存.

```shell
# 应用缓存和窗口缓存的比例 应用缓存=buff/(2^tcp_adv_win_scale)
net.ipv4.tcp_adv_win_scale=1 
```

TCP小包优化: 

SWS

SWS避免方案
1. 接收方当发现窗口移动小于 min(mss,缓存/2时)通知窗口为0
2. Nagle算法. 当已发送未确认为0时发送或者数据长度>mss时发送.

TCP延迟确认.

TCP_CORK


## tcp缓冲区

每一个tcp连接需要4k内存，在连接传输数据的时候也需要内存. 在free buff/cache中有所体现.
每个连接在内核里都有内存缓冲区，如果配置小就会传输慢，配置太大又会增大内存.
TCP是面向流传输的、顺序的、可靠传输协议，他使用确认机制来确保报文发送成功，所有发送的报文必须有ACK.
如果报文在一段时间内没有ACK（RTO），才会删除
发送方的发送速率取决于接受方的的buffer大小.

BDP决定了能够发送的最大的报文.
BDP(Bandwidth Delay Product)=带宽*延迟
根据BDP调整相应的buffer大小，应该让发送buffer的最大值=BDP.
每秒能发送的数据量 = 发送窗口大小 * 1000 / 延迟
可以直接设置socket的SO_SNDBUF和SO_RCVBUF，但是千万不要在 socket 上直接设置 SO_SNDBUF或者 SO_RCVBUF，这样会关闭缓冲区的动态调整功能。
如果设置了SO_SNDBUF和SO_RCVBUF，需要调整以下2个参数. 实际的带宽是 min(SO_SNDBUF,wmem_max)
net.core.rmem_max = 212992
net.core.wmem_max = 212992

## TCP性能和发送接收Buffer的关系

```shell
sudo sysctl -a | egrep "mem|adv_win|moderate|backlog"
net.core.rmem_default = 212992
net.core.rmem_max = 212992
net.core.wmem_default = 212992
net.core.wmem_max = 212992
net.ipv4.tcp_adv_win_scale = 1
net.ipv4.tcp_moderate_rcvbuf = 1
net.ipv4.tcp_rmem = 4096 87380 6291456
net.ipv4.tcp_wmem = 4096 16384 4194304
net.ipv4.udp_rmem_min = 4096
net.ipv4.udp_wmem_min = 4096
vm.lowmem_reserve_ratio = 256 256 32

```

# 建联相关

# 内网快速失败

sysctl -w net.ipv4.tcp_syn_retries=2
sysctl -w net.ipv4.tcp_synack_retries=2
sysctl -w net.core.netdev_max_backlog=32768
sysctl -w net.ipv4.tcp_max_syn_backlog=32768
sysctl -w net.ipv4.tcp_fastopen=3

sysctl -w net.core.rmem_default=10485760
sysctl -w net.core.rmem_max=31457280
sysctl -w net.core.wmem_default=10485760
sysctl -w net.core.wmem_max=31457280

# 0.5 0.66 1

sysctl -w net.ipv4.tcp_mem='3027648 3996495 6055296'
sysctl -w net.ipv4.tcp_rmem='4096 212992 31457280'
sysctl -w net.ipv4.tcp_wmem='4096 212992 31457280'

``` shell
# 扩展窗口
net.ipv4.tcp_window_scaling = 1

# 发送缓冲区
# 依赖待发送的数据动态调节
net.ipv4.tcp_wmem=4096 16384 4194304

# 接受缓冲区
# 依据空闲系统内存的数量来调节接收窗口
net.ipv4.tcp_rmem = 4096 87380 6291456
net.ipv4.tcp_moderate_rcvbuf = 1
net.ipv4.tcp_mem = 88560 118080 177120

# 使用TCPtimestamp解决 PAWS以及计算更精准的RTO
 
```

ss -itmpn 'sport == '9092''