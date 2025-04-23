---
title: Stress Test Tools
date: 2025-04-08 09:43:05
tags: tools
categories: Posts
excerpt: 压力测试工具介绍
---

## 查看磁盘类型

```bash
# 1 HDD, 0 SSD
cat /sys/block/sdX/queue/rotational
lsblk -d -o name,rota,model
```

## fio

Linux磁盘IO测试工具

```bash
# 顺序读取
fio --name=seqread --filename=testfile --rw=read --bs=1M --size=2G --numjobs=1 --iodepth=1 --runtime=60 --time_based --direct=1 --group_reporting
# 顺序写入
fio --name=seqwrite --filename=testfile --rw=write --bs=1M --size=2G --numjobs=1 --iodepth=1 --runtime=60 --direct=1 --time_based --group_reporting

# 随机读取
fio --name=randread --filename=testfile --rw=randread --bs=4k --size=2G --numjobs=4 --iodepth=16 --runtime=60 --direct=1 --time_based --group_reporting
# 随机写入
fio --name=randwrite --filename=testfile --rw=randwrite --bs=4k --size=2G --numjobs=4 --iodepth=16 --runtime=60 --direct=1 --time_based --group_reporting

# 高并发随机读写
fio --name=randrw --filename=testfile --rw=randrw --rwmixread=70 --bs=4k --size=2G --numjobs=8 --iodepth=16 --direct=1 --runtime=60 --time_based --group_reporting
```

--bs=4k：每次 I/O 4KB

--rw=：读写模式（read、write、randread、randwrite、randrw）

--numjobs=4：并发线程数

--direct=1：绕过缓存，直接访问设备

--ioengine=libaio：使用异步 I/O 引擎 或--ioengine=io_uring （5.1 内核以上）更能反映ssd盘的真实性能

--runtime=30s：测试持续 30 秒

## DiskSpd

Windows的磁盘IO测试工具

https://github.com/microsoft/diskspd

```cmd
diskspd.exe -c64M -b64K -t1 -o32 -si64K -w50 -S D:\test
```

## prime95

windows CPU测试工具

https://prime95.net/download/

## iperf3

网络测试工具，linux和windows都可使用

```bash
# 服务器端
iperf3 -s
# 设置监控时间间隔为 10 秒，监控端口为 5201
iperf3 -s -i 10 -p 5201
# 客户端
iperf3 -c $IP -R
# 指定-c测速服务器IP，-p指定端口为5201，-t测速时间5s，-P指定发送连接数10，-R表示下载测速
iperf3 -c $IP -p 5201 -t 5 -P 10 -R
# 指定-c测速服务器IP，-t测速时间20s，-i指定监控时间间隔为5s
iperf3 -c 43.248.136.69 -t 20 -i 5
# 指定-c测速服务器IP，-i指定监控时间间隔为7s，-n指定发送数据量为5G
iperf3 -c 43.248.136.69 -i 7 -n 5G
# 指定-c测速服务器IP，-i指定监控时间间隔为2s，-F指定要传输的文件，-t指定测速时间20s
iperf3 -c 43.248.136.69 -i 2 -F Python-3.7.1rc2.tgz -t 20
```

## iostat

Linux 磁盘监控工具

```bash
# 每 1 秒采样一次，持续 5 秒
iostat -xd sda 1 5
```

## nmon 监控

Linux 系统资源监控工具。

可以收集服务器系统资源使用情况，并输出成特定文件，且可利用excel分析工具进行数据统计分析

```bash
# 每 5 秒采样一次，持续 120 秒
nmon -f -s 5 -c 120 -m /tmp/nmon.out
```

|参数|用法|
|--|--|
|c|带条形图的 CPU 利用率统计信息（CPU 核心线程）|
|m|内存和交换统计|
|d|磁盘 I/O 繁忙百分比 & 每秒读\写数据量 KB/s 图|
|r|资源：机器类型，名称，缓存详细信息和操作系统版本以及 Distro + LPAR|
|t|top 进程，1 基础、3 性能、4 大小、5 I/O 仅 root 用户可用|
|n|网络统计信息和错误（如果没有错误，则消失）|
|j|文件系统，包括日记文件系统|
|k|内核统计信息运行队列，上下文切换，派生，平均负载和正常运行时间|
|U|CPU 使用率统计信息 user, user_nice, system, idle, iowait, irq, softirq, steal, guest, guest_nice|
|u|进程详细信息|
