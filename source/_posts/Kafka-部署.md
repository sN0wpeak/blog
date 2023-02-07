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
4. 网络
根据实际流量计算.

## 集群规模计算

预估集群规模和容量需要根据实际流量进行分配，所以需要提供以下信息才能准确的进行容量预估

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

磁盘

- Writes: W * R
- Reads: ( R + C - 1) * W
- Disk Size (GB)= max(best_disk(msg_throughput_pd, writes), MIN_DISK_G) 最少200G
- Disk Throughput (Read + Write): W * R + L * W

带宽

- Network Read Throughput: (R + C - 1 ) * W
- Network Write Throughput: W * R
- Network Throughput = Network Read Throughput + Network Write Throughput

分区数
为最佳性能的指导方针，每个代理的分区不应超过 4000 个，集群中的分区不应超过 200,000 个。

## 系统参数

``` properties
# net
net.ipv4.tcp_adv_win_scale = 1
net.ipv4.tcp_moderate_rcvbuf = 1
net.ipv4.tcp_rmem = 4096    87380   6291456
net.ipv4.tcp_wmem = 4096    16384   4194304
net.ipv4.tcp_max_syn_backlog = 1000
net.core.netdev_max_backlog = 1024

# 磁盘
vm.max_map_count=655350
vm.overcommit_memory = 0

cat /proc/vmstat | egrep "dirty|writeback" 
nr_dirty 21845
nr_writeback 0
nr_writeback_temp 0
nr_dirty_threshold 32715981 
nr_dirty_background_threshold 2726331
```

## broker参数

``` properties
# buff大小，如果需要吞吐比较大的话可以往大调. 最大10485760 
# 操作系统会自动调节 一般不用设置，除非你知道这是最优值
# receive.buffer.bytes=10485760       
# buff大小，如果需要吞吐比较大的话可以往大调. 最大10485760        
# 操作系统会自动调节 一般不用设置，除非你知道这是最优值
# send.buffer.bytes=10485760
# 安全协议
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="WL1Lv8nOnezo6uw5";
replication.factor=3
delete.topic.enable=false
auto.leader.rebalance.enable=false
auto.create.topics.enable=false
num.recovery.threads.per.data.dir=8
log.dirs=
broker.id=
listeners=
```

## JVM

注重延迟的话可以直接用Java17 ZGC就行，不过会影响吞吐。
一般的话G1就可以了，配置如下.

* MaxGCPauseMillis 每次希望GC完成的时间. 默认200ms，可以设置成20ms，减少卡顿.
* InitiatingHeapOccupancyPercent. GC启动的heap百分比.

```properties
# export KAFKA_JVM_PERFORMANCE_OPTS="-server -Xmx6g -Xms6g -XX:MetaspaceSize=96m -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M -XX:MinMetaspaceFreeRatio=50 -XX:MaxMetaspaceFreeRatio=80 -XX:+ExplicitGCInvokesConcurrent" # /usr/local/kafka/bin/kafka-server-start.sh -daemon /usr/local/kafka/config/server.properties
```

# 参考

https://kafka.apache.org/documentation/
《kafka权威指南》