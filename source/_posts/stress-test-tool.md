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
# 测试写性能
fio --name=write_test --filename=testfile --size=1G --bs=4k --rw=write --ioengine=libaio --direct=1 --numjobs=4 --time_based --runtime=30s
# 测试读性能
fio --name=read_test --filename=testfile --size=1G --bs=4k --rw=read --ioengine=libaio --direct=1 --numjobs=4 --time_based --runtime=30s
# 测试读写混合性能
fio --name=randrw_test --filename=testfile --size=1G --bs=4k --rw=randrw --rwmixread=70 --ioengine=libaio --direct=1 --numjobs=4 --time_based --runtime=30s
```

--bs=4k：每次 I/O 4KB

--rw=：读写模式（read、write、randread、randwrite、randrw）

--numjobs=4：并发线程数

--direct=1：绕过缓存，直接访问设备

--ioengine=libaio：使用异步 I/O 引擎

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
iostat -dx sda 1 5
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
