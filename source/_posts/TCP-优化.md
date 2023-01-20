---
title: TCP-优化
date: 2023-01-17 13:09:15
tags: tcp
---


# 简介


## tcp缓冲区
每一个tcp连接需要4k内存，在连接传输数据的时候也需要内存. 在free buff/cache中有所体现. 每个连接在内核里都有内存缓冲区，如果配置小就会传输慢，配置太大又会增大内存.
TCP是面向流传输的、顺序的、可靠传输协议，他使用确认机制来确保报文发送成功，所有发送的报文必须有ACK. 如果报文在一段时间内没有ACK（RTO），才会删除
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


sudo sysctl -a | egrep "rmem|wmem|adv_win|moderate"
net.core.rmem_default = 212992
net.core.rmem_max = 212992
net.core.wmem_default = 212992
net.core.wmem_max = 212992
net.ipv4.tcp_adv_win_scale = 1
net.ipv4.tcp_moderate_rcvbuf = 1
net.ipv4.tcp_rmem = 4096    87380   6291456
net.ipv4.tcp_wmem = 4096    16384   4194304
net.ipv4.udp_rmem_min = 4096
net.ipv4.udp_wmem_min = 4096
vm.lowmem_reserve_ratio = 256   256 32


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
```
