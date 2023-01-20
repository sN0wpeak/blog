---
title: Kafka-部署
date: 2023-01-20 09:35:27
tags: kafka
---

# 安装方案

看官方安装吧，网上也有k8s方案和ansible方案，这里不在多讲了。我们主要说下怎么做优化。
https://kafka.apache.org/quickstart
https://github.com/sleighzy/ansible-kafka
https://github.com/strimzi/strimzi-kafka-operator

# 部署优化

## 硬件选择:

1. 磁盘吞吐
   在线服务最好选择SSD吧，因为要提供更低的延迟. 大数据服务可以选择HDD. 磁盘类型最好选择xfs.

2. 容量
   取决于保存时间和每天的消息量. 原则上最好有些额外的buffer，buffer的大小取决于是否有突发流量以及这些预留的buffer到磁盘满的时间（你需要至少多久扩容？）.
   一般建议80%
   Disk Size = 每天消息量 * 保存天数 + 预留buffer
3. CPU和内存
   cpu和内存建议1:4即可. kafka对内存依赖主要是做log的page cache缓存，一般只缓存partition active文件就可以了.
   堆内存也不需要太大一般5G就够了，因为服务端基本上都是零拷贝（除非有不兼容需要转换的消息），不需要解析消息的.

## 集群规模计算

预估集群规模和容量需要根据实际流量进行分配。所以需要提供以下信息才能准确的进行容量预估

* 新增消息数
* 消息留存时间
* 平均消息大小
* 备份数
* 是否启用压缩
* 分区数

定义一些变量，用于后面公式计算

* W - MB/sec每秒写入的数据
* R - 副本数
* C - consumergroup的数量
* RT - 消息保存时间秒

机型
机型主要是按吞吐来算的. 各个机型的吞吐如下
TODO 给出linger.ms=5 or 10ms 的最高吞吐

- m6i.xlarge: min(50000 * 1024, ebs_bandwidth)
- m6i.2xlarge: min(100000 * 1024, ebs_bandwidth)
- m6i.4xlarge: min(200000 * 1024, ebs_bandwidth)

磁盘

- Writes: W * R
- Reads: ( R + C - 1) * W
- Disk Size (GB)= max(best_disk(msg_throughput_pd, writes), MIN_DISK_G) 最少200G
- Disk Throughput (Read + Write): W * R + L * W
- IOPS和吞吐的关系是10:1
- 写入吞吐最低200M

带宽

- Network Read Throughput: (R + C - 1 ) * W
- Network Write Throughput: W * R
- Network Throughput = Network Read Throughput + Network Write Throughput

分区数
为最佳性能的指导方针，每个代理的分区不应超过 4000 个，集群中的分区不应超过 200,000 个。

代码

``` python
import math

MAX_DISK_THROUGHPUT_M = 1000
MAX_DISK_IOPS = 10000
# IOPS和吞吐的比例.
IOPS_DISK_THROUGHPUT_SCALA = 12
MIN_DISK_THROUGHPUT_M = 200
MIN_DISK_IOPS = 2000
UNIT_SECOND = 24 * 60 * 60
UNIT_1K = 1024
UNIT_1M = 1 * UNIT_1K * UNIT_1K
MIN_DISK_G = 200
UNIT_1G = 1 * UNIT_1M * UNIT_1K
UNIT_1T = 1 * UNIT_1G * UNIT_1K


class KafkaMachine:
def __init__(self, instance_types="m6i.4xlarge", cpu=16, mem=64,
net_bandwidth=12.5 * UNIT_1G / 8, io_bandwidth=10.0 * UNIT_1G / 8, throughput=100000 * 1024):
self.instance_types = instance_types
self.cpu = cpu
self.mem = mem
self.net_bandwidth = net_bandwidth * 0.7
self.io_bandwidth = io_bandwidth * 0.7
self.throughput = min(throughput, self.io_bandwidth)


# 目前只选择m6i机型
KAFKA_MACHINES = [
KafkaMachine(instance_types="m6i.xlarge", cpu=4, mem=16,
net_bandwidth=12.5 * UNIT_1G / 8, io_bandwidth=10.0 * UNIT_1G / 8, throughput=50000 * 1024),
KafkaMachine(instance_types="m6i.2xlarge", cpu=8, mem=32,
net_bandwidth=12.5 * UNIT_1G / 8, io_bandwidth=10.0 * UNIT_1G / 8, throughput=100000 * 1024),
KafkaMachine(instance_types="m6i.4xlarge", cpu=16, mem=64,
net_bawndwidth=12.5 * UNIT_1G / 8, io_bandwidth=10.0 * UNIT_1G / 8, throughput=200000 * 1024),
]


def match_machine(w):
best_template = None
for m in KAFKA_MACHINES:
if w < m.throughput:
best_template = m
break
if not best_template:
best_template = KAFKA_MACHINES[-1]
return best_template


def compute():
    # 基础数据计算
    msg_throughput_ps = 0
    msg_throughput_pd = 0
    if msg_throughput_pd == 0:
        msg_throughput_pd = msg_count_per_day * msg_size
    if msg_throughput_ps == 0:
        msg_throughput_ps = msg_size * qps
    msg_throughput_ps = msg_throughput_ps * reserve_percentage
    msg_throughput_pd = msg_throughput_pd * reserve_percentage
    w = msg_throughput_ps
    reads = (replica + consumer_group_count - 1) * w
    writes = w * replica
    network_read_throughput = (replica + consumer_group_count - 1) * w
    network_write_throughput = w * replica
    network_throughput = network_read_throughput + network_write_throughput

    # 机器计算
    machine = match_machine(w)
    machine_num = max(1, math.ceil(w / machine.throughput)) * az_size

    # 磁盘计算
    disk = max(best_disk(msg_throughput_pd, writes), MIN_DISK_G)
    disk_scala = best_disk_scala(writes)
    disk_iops = min(max(writes * disk_scala * IOPS_DISK_THROUGHPUT_SCALA / UNIT_1M, MIN_DISK_IOPS), MAX_DISK_IOPS)
    disk_throughput_m = min(max(writes * disk_scala / UNIT_1M, MIN_DISK_THROUGHPUT_M), MAX_DISK_THROUGHPUT_M)

    # 输出
    print(f"""应用名称: {app_name}
环境: {env}
ZK: (2C,4G) * 3 (可共用通业务线)
Kafka: ({machine.instance_types}({machine.cpu}C,{machine.mem}G) 硬盘 gp3: {disk_iops} IOPS、{disk_throughput_m} MiB/s) * {machine_num}

Details:
几倍预留: {reserve_percentage - 1}
IO Write Throughput: {writes / UNIT_1M:.2f}MB
IO Read Throughput: {reads / UNIT_1M:.2f}MB
Total Disk Usage: {disk:.2f}GB
Per Disk Usage: {disk / machine_num:.2f}GB
Network Throughput: {network_throughput / UNIT_1M:.2f}MB
Network Read Throughput: {network_read_throughput / UNIT_1M:.2f}MB
Network Write Throughput: {network_write_throughput / UNIT_1M:.2f}MB
""")


def best_disk(msg_throughput_pd, writes):
if msg_throughput_pd > 0:
return relation_day * msg_throughput_pd * replica / UNIT_1G
else:
return writes * relation_day * UNIT_SECOND / UNIT_1G


def best_disk_scala(writes):
if writes / UNIT_1M < 100:
return 3
else:
return 2


if __name__ == '__main__':
# 预留百分比，通常保留3倍.
reserve_percentage = 1
# 应用名称
app_name = 'sbu-spot-binlog-prod'
# 环境
env = 'prod'
# 每秒峰值
qps = 100000
# 单个消息大小
msg_size = 4096

    latencyType = '正常'
    # az数
    az_size = 3
    # 保存天数
    relation_day = 7
    # 多少个消费者
    consumer_group_count = 3
    # 副本数
    replica = 3
    # 每天的消息量可选
    msg_count_per_day = 0

    compute()
```

## 系统参数


``` shell
# net
net.ipv4.tcp_adv_win_scale = 1
net.ipv4.tcp_moderate_rcvbuf = 1
net.ipv4.tcp_rmem = 4096    87380   6291456
net.ipv4.tcp_wmem = 4096    16384   4194304
net.ipv4.tcp_max_syn_backlog = 1000
net.core.netdev_max_backlog = 1024

# 磁盘
cat /proc/vmstat |
```


## JVM
注重延迟的话可以直接用Java17 ZGC就行，不过会影响吞吐。