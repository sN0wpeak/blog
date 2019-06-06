---
title: linux性能优化实战-CPU
date: 2019-06-06 17:13:20
tags: linux, 性能优化, 故障排查
---

# 负载
当我们发现系统很慢的时候，通过会查看系统的负载。 一般来讲会通过 top或这是 uptime命令

``` shell
 top
top - 17:15:48 up 622 days,  4:51,  9 users,  load average: 5.01, 5.02, 5.05
Tasks: 210 total,   1 running, 205 sleeping,   1 stopped,   3 zombie
%Cpu(s):  0.3 us,  0.3 sy,  0.0 ni, 98.7 id,  0.8 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  8010528 total,   179432 free,  4879284 used,  2951812 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  2776264 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
22239 root      20   0 5652808 140536   4732 S   0.7  1.8  18:21.59 java
 6731 root      20   0 5657940 123540   3640 S   0.3  1.5 112:09.73 java
10219 redis     20   0  474672 336468    384 S   0.3  4.2 315:43.34 redis-server
10274 root      20   0  571320  15488   2484 S   0.3  0.2  14:07.64 YDService
24606 root      20   0       0      0      0 S   0.3  0.0   0:00.54 kworker/0:0

uptime
 17:16:22 up 622 days,  4:51,  9 users,  load average: 5.05, 5.04, 5.05
```

上边的load average: 5.05, 5.04, 5.05 就是系统的平均负载了。 分别代表过去1分钟，5分钟，15分钟的平均负载。 这给我们提供了分析系统负载趋势的数据源。 
可以观察 过去15分钟内的负载的变化

那什么是平均负载呢？
``` shell
man uptime
System load averages is the average number of processes that are either in a runnable or uninterruptable state.  A process in a runnable state is either using the CPU or waiting  to  use the CPU.   
A process in uninterruptable state is waiting for some I/O access, eg waiting for disk.  The averages are taken over the three time intervals.  Load averages are not normalized for the number of CPUs in a system, so a load average of 1 means a single CPU system is loaded all the time while on a 4 CPU system it means it was idle 75% of the time.
```
平均负载就是单位时间里，进程状态为runnable(状态为R Running or Runnable)或者是uninterruptable(状态为D Disk sleep)的进程数。
不可中断是操作系统对于硬件的一种保护措施，是内核态执行的。

如果你有4个CPU，如果负载是4 那就说明CPU全部被占满了，资源得到了最大的利用。

``` shell
查看cpu数
grep 'model name' /proc/cpuinfo | wc -l
2
```

> 当平均负载在70%CPU数量的时候比较合适

负载高不代表CPU的使用率高。
* CPU密集型的程序，负载应该和CPU数一致
* IO密集型的进程，IO负载会高，CPU使用率不一定会高
* 大量等待CPU调度的进程也会造成load变高，CPU也会随之升高

apt install systat

常用的一些命令包括 top, mpstat, pidstat, uptime

``` shell
mpstat -P  ALL 1 -u
Linux 3.10.0-514.26.2.el7.x86_64 (tx-relay1.common.bj.tx.lan) 	2019年06月06日 	_x86_64_	(4 CPU)

18时08分38秒  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
18时08分39秒  all   42.86    0.00   21.94    0.26    0.00    0.26    0.00    0.00    0.00   34.69
18时08分39秒    0   38.14    0.00   21.65    0.00    0.00    0.00    0.00    0.00    0.00   40.21
18时08分39秒    1   37.37    0.00   21.21    0.00    0.00    1.01    0.00    0.00    0.00   40.40
18时08分39秒    2   47.96    0.00   20.41    0.00    0.00    0.00    0.00    0.00    0.00   31.63
18时08分39秒    3   47.47    0.00   24.24    2.02    0.00    0.00    0.00    0.00    0.00   26.26

pidstat -u -p  25397
Linux 3.10.0-514.26.2.el7.x86_64 (tx-relay1.common.bj.tx.lan) 	2019年06月06日 	_x86_64_	(4 CPU)

18时14分24秒   UID       PID    %usr %system  %guest    %CPU   CPU  Command
18时14分24秒  1038     25397    0.00    0.00    0.00    0.00     2  python
```


# CPU上下文切换

# 短时进程

# 大量不可中断进程

# 大量僵尸进程

# linux软中断
中断是指操作系统响应硬件请求的一种机制，它会打断正常的进程正常的调度，然后调用内核中的中断处理程序来处理中断。 中断是一种异步的事件处理机制，可以提高系统的并发性





