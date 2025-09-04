---
title: Linux Kernel Tuning
date: 2025-05-14 09:12:14
tags: linux
categories: Notes
excerpt: 内核参数优化
---

## 性能指标

- 进程指标
  - 程序与进程
  - 进程与线程
  - 进程优先级和nice级别
    - nice级别修改进程的静态优先级，由低到高，从19到-20
- 内存指标
  - Linux在负载比较大（内存很紧张）的时候一般会看到两个进程：kswapd0和kswapd1
  - kswapd进程如果频繁被唤醒会过度消耗CPU，此时可以通过设置大页内存（HugePages）来解决
- 文件系统指标
  - EXT3/EXT4/XFS文件系统特性
  - 主流的文件系统是EXT4和XFS
  - 文件系统选择
    - web类应用
      - 读操作频繁，写操作一般，两者皆可
    - 数据库类
      - 写操作频繁的结构化数据库类应用，xfs最佳
    - 普通应用类场景
  - 磁盘存储使用raid技术，优先使用硬raid
    - 写频繁，数据安全性要求一般，可用raid0
    - 安全性要求高，可用raid1或raid10
  - 合理分区有助于提高文件系统性能
- 磁盘I/O指标
  - 电梯算法
  - 当进程尝试改变数据时，会首先修改内存中的数据，此时磁盘和内存中的数据就不一致了，内存中的数据就叫脏缓冲
- 网络指标
  - 拓扑结构
  - 硬件网卡
  - 内核网络参数

## 调优分析工具

### CPU

- vmstat
  - `-n` 只显示一次表头
  - `vmstat -n 1 5` 每1秒显示一次，共显示5次
  - proc 队列和等待状态
    - r 表示运行和等待CPU时间片的进程数，如果长期大于系统CPU的个数，说明CPU不足，需要增加CPU。
    - b 表示在等待资源的进程数，如正在等待I/O、内存交换等。
  - memory
    - swpd列表示切换到内存交换区的内存数量，以KB为单位
    - free列表示当前空闲的物理内存数量，以KB为单位
    - buff列表示buffers cache的内存数量，一般对块设备的读写才需要缓冲
    - cache列表示page cached的内存数量，一般作为文件系统cached，频繁访问的文件都会被cached
  - swap
    - si列表示由磁盘调入内存，也就是内存进入内存交换区的数量
    - so列表示由内存调入磁盘，也就是内存交换区进入内存的数量
    - 一般情况下，si、so的值都为0，如果si、so的值长期不为0，则表示系统内存不足
  - io
    - bi列表示从块设备读入数据的总量（即读磁盘）​，KB/s
    - bo列表示写入到块设备的数据总量（即写磁盘）​，KB/s
  - system
    - in 表示在某一时间间隔中观测到的每秒设备中断数
    - cs 列表示每秒产生的上下文切换次数
  - cpu
    - us列显示了用户进程消耗的CPU时间百分比
    - sy列显示了内核进程消耗的CPU时间百分比
    - id列显示了CPU处在空闲状态的时间百分比
    - wa列显示了I/O等待所占用的CPU时间百分比
  > 重点注意procs项r列的值和CPU项us、sy和id列的值
- uptime
  - 1min内、5min内、15min内
  - load average这个输出值，这3个值一般不能大于系统CPU的个数
- mpstat
  - 可以查看多核CPU中每个计算核的统计数据
  - `mpstat -P ALL 1 5` 每1秒显示一次，共显示5次

### 内存

- free
  - Used + free + buff/cache = total
  - available是在buff/cache基础上减去了shared以及buffer内存损耗剩下的内存资源
  - 查看内存是否充足，只需要关注available一列即可
- smem
  - RSS（Resident Set Size）​：使用top命令可以查询到，是最常用的内存指标，表示进程占用的物理内存大小
  - PSS（Proportional Set Size）​：所有使用某共享库的程序均分该共享库占用的内存
  - USS（Unique Set Size）​：进程独自占用的内存，它只计算了进程独自占用的内存大小，不包含任何共享的部分
  - `smem -k -s uss` -k参数用来显示内存单位，-s对uss列排序， -p百分比形式显示
  - `smem -u -k` 显示每个用户的内存使用情况
  - `smem -k -P prometheus` 显示prometheus进程的内存使用情况

### 磁盘

- iotop
  - `-u` 指定用户名
  - `-p` 指定进程id
  - `-P` 只显示进程，默认显示所有的线程
  - `-k` 字节显示
  - `-t` 每一行前添加一个当前时间
- iostat
  - `iostat-d 3 5` 命令组合查看系统磁盘的使用状况
  - 第一项是系统从启动以来到统计时的所有传输信息，从第二次输出的数据才代表在检测的时间段内系统的传输值
  - 如果KB_wrtn/s值很大，表示磁盘的写操作很频繁，可以考虑优化磁盘或者优化程序
  - 如果KB_read/s值很大，表示磁盘直接读取操作很多，可以将读取的数据放入内存中进行操作
  - `-x` 显示扩展信息
  
### 网络

- ping
- traceroute
  - `traceroute -i eth0 -s 192.168.1.1 -w 10 www.baidu.com 100`
  - 定eth0网络接口发送数据包，同时指定本地发送数据包的IP为192.168.1.1，并设置超时时间为10s，最后设置发送数据包的大小为100Byte
  - 由于traceroute是利用ICMP连接的，有些网络设备（如防火墙）可能会屏蔽ICMP通过的权限，因此也会出现节点没有回应的状态
- mtr
  - `mtr www.baidu.com`
  
### 综合工具

- htop

## 分析性能瓶颈

出现CPU瓶颈有两种原因：一种是应用程序bug导致CPU资源耗尽；另一种是CPU资源确实不足。实际应用中，第一种情况更常见，此时需要从应用程序角度来排查问题。而如果出现的是第二种情况，就要考虑增加CPU资源了。

通过内存监控工具free、top、smem等获取内存指标。如果发现可用内存（available）持续减少，系统进程中kswapd进程频繁出现，同时Swap交换空间占用率持续增高，那么基本可以判断是系统内存资源不足了

通过磁盘I/O监控命令，如iostat、iotop等，发现%iowait持续超过80%，并且CPU的I/O等待也非常高。应用系统读、写操作响应时间很长，或网络利用率（网络正常情况下）非常低。磁盘请求队列（/sys/block/sda/queue/nr_requests）变满，处理请求时间变长

网络瓶颈表现得比较直接，例如，连接服务器变慢，但服务器正常，大部分连接请求超时，网络传输带宽上不去等，服务器的CPU、内存、磁盘I/O都表现正常，但是操作系统上面的业务系统访问却很缓慢，这都是典型的网络带宽瓶颈

## 调优

### 系统安装和分区优化

1. 磁盘与RAID

    实际应用领域中使用最多的RAID等级是RAID0、RAID1、RAID5、RAID10

    |级别|读写性能|安全性|磁盘利用率|成本|场景|
    |-|-|-|-|-|-|
    |RAID0|最好|最差|最高|最低|对安全性要求不高，大文件写的系统|
    |RAID1|读和单个磁盘无分别，写则同时写|最高|低 50%|较高|适用存放重要数据，如数据库类应用|
    |RAID5|读：raid5=raid0<br>写：raid5小于单个磁盘写|raid5<raid1|raid5>raid1|中等|存储性能、安全性、成本兼顾|
    |RAID10|读：raid10=raid0<br>写：raid10=raid1|raid10=raid1|raid10=raid1 50%|较高|集合了raid1和raid0的优点，但磁盘利用率50%|

    部署线上服务器的时候，最好配置两组RAID，一组是系统盘RAID，对系统盘（安装操作系统的磁盘）推荐配置为RAID1，另一组是数据盘RAID，对数据盘（存放应用程序、各种数据）推荐采用RAID1、RAID5或者RAID10

2. 系统版本选择

    推荐centos发型版本

3. 分区与swap

    系统分区 raid1

    数据分区 raid1 raid5

    默认安装会使用LVM（逻辑卷管理）进行分区管理。作为线上生产环境，强烈不推荐使用LVM，因为LVM的动态扩容功能，对现在大硬盘时代来说，基本没什么用处了。

    一般可以一次性规划好硬盘的最大使用空间，相反，使用LVM带来的负面影响更大，首先，它影响磁盘读写性能，其次，它不便于后期的运维。因为LVM的磁盘分区一旦故障，数据基本无法恢复。

    swap 可以一定程度上防止内存不够时触发oom，虽然某些业务系统，如redis、elasticsearch可能使用swap会导致性能下降，可以通过设置/proc/sys/vm/swappiness，调整swap使用概率

    物理内存在16GB以下的，Swap设置为物理内存的2倍即可；而物理内存大于16GB的话，一般推荐swap设置8GB左右即可。

    > 但在工作中大部分见到的是使用lvm，以及去掉swap。

### 系统资源参数优化

ulimit -a可以看到所有系统资源参数，这里面需要重点设置的是open files和max user processes

---
历史命令记录，将代码加入到/etc/profile中

```bash
#history
USER_IP=`who -u am i 2>/dev/null| awk '{print $NF}' | sed -e 's/[()]//g'
HISTDIR=/usr/share/.history
if [ -z ŞUSER IP ]
then
USER IP=`hostname
fi
if [ ! -d SHISTDIR ]
then
mkdir -p $HISTDIR
chmod 777 $HISTDIR
fi
if [ ! -d $HISTDIR/${LOGNAME} ]
then
mkdir -p SHISTDIR/${LOGNAME}
chmod 300 SHISTDIR/${LOGNAME}
fi
export HISTSIZE=4000
DT=`date +%Y%m%d_%H%M%S`
export HISTFILE="$HISTDIR/${LOGNAME}/${USER_IP}.history.$DT"
export HISTTIMEFORMAT="[%Y.%m.%d %H:%M:%S]"
chmod 600 $HISTDIR/${LOGNAME}/*.history* 2>/dev/null
```

将每个用户的shell命令执行历史以文件的形式保存在/usr/share/.history目录中，每个用户一个文件夹，并且文件夹下的每个文件以IP地址加shell命令操作时间的格式命名

---

### 内核参数调优

Linux提供了/proc这样一个虚拟文件系统，通过它在Linux内核空间和用户间之间进行通信。在/proc文件系统中，可以将对虚拟文件的读写作为与内核中实体进行通信的一种手段，但是与普通文件不同的是，这些虚拟文件的内容都是动态创建的。这些文件虽然使用查看命令查看时会返回大量信息，但文件本身的大小却显示为0字节，因为这些都是驻留在内存中的文件

- /proc/sys/net是跟网络相关的内核参数。
- /proc/sys/kernel是跟内核相关的内核参数。
- /proc/sys/vm是跟内存相关的内核参数。
- /proc/sys/fs是跟文件系统相关的内核参数。

开启Linux代理转发功能，可以直接修改内核参数ip_forward对应在/proc下的文件/proc/sys/net/ipv4/ip_forward

```bash
# 查看
cat /proc/sys/net/ipv4/ip_forward
# 修改
echo "1" > /proc/sys/net/ipv4/ip_forward
```

echo命令不能检查参数的一致性；其次，echo方式修改的内核参数在重启系统之后，所有的内核修改都会丢失

使用 sysctl修改内核参数

```bash
# 查看
sysctl kernel.msgmnb
sysctl net.ipv4.tcp_syn_retries
# 写入，但是重启后也会失效
sysctl -w kernel.msgmnb=40960
```

永久修改编辑sysctl.conf文件，添加`sysctl kernel.msgmnb=40960`，然后 `sysctl -p` 立即生效配置

### 网络内核参数优化

- `/proc/sys/net/ipv4/tcp_syn_retries`
  - 表示对于一个新建连接，内核要发送多少个SYN连接请求才决定放弃
  - 默认值是5，建议设置为2
- `/proc/sys/net/ipv4/tcp_keepalive_time`
  - 表示当keepalive启用的时候，TCP发送keepalive消息的频度
  - 默认值是7200s，建议修改为300s
- `/proc/sys/net/ipv4/tcp_orphan_retries`
  - 参数表示孤儿Socket废弃前重试的次数，重负载Web服务器建议调小
- `/proc/sys/net/ipv4/tcp_syncookies`
  - 表示开启SYN Cookies。当出现SYN等待队列溢出时，启用Cookies来处理，可防范少量SYN攻击
  - 默认值是0
- `/proc/sys/net/ipv4/tcp_max_syn_backlog`
  - 表示设置SYN队列最大长度
  - 默认值为1024，加大队列长度为8192，可以容纳更多等待连接的网络连接数
- `/proc/sys/net/ipv4/tcp_synack_retries`
  - 用来降低服务器SYN+ACK报文重试次数，尽快释放等待资源
  - 默认是5次
  - 此参数是决定内核在放弃连接之前所送出的SYN+ACK的数目

  > 针对SYN攻击，可以启用SYN Cookie、设置SYN队列最大长度以及设置SYN+ACK最大重试次数。设置tcp_syncookies为1就是启用了SYN Cookie

- `/proc/sys/net/ipv4/tcp_tw_recycle`
  - 表示开启TCP连接中TIME-WAIT sockets的快速回收
  - 默认值为0，表示关闭
  - 普通Web服务器建议设置为`echo 1>/proc/sys/net/ipv4/tcp_tw_recycle`

  > 从 Linux 内核 4.12 开始，tcp_tw_recycle 参数已经被彻底移除
  >
  > 查看TIME-WAIT数量
  > netstat -nat | awk '{print $6}' | sort | uniq -c | sort -rn

- `/proc/sys/net/ipv4/tcp_tw_reuse`
  - 表示开启重用，允许将TIME-WAIT sockets重新用于新的TCP连接
  - 此值默认值为0，表示关闭；启用该resuse的同时，必须同时启用快速回收recycle（即tcp_tw_recycle为1）​。
- `/proc/sys/net/ipv4/tcp_fin_timeout`
  - 表示处于TIME_WAIT状态的连接在回收前必须等待的最小时间
  - 普通web服务器建议设置为15秒
- `/proc/sys/net/ipv4/tcp_keepalive_probes`
  - 用来减少超时前的探测次数，普通Web服务器建议设置为 5
- `/proc/sys/net/core/netdev_max_backlog`
  - 用来设置每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目
  - 默认值1000，建议3000
- `/proc/sys/net/core/rmem_max、wmem_max`
  - 可以提高TCP的最大缓冲区大小。
  - rmem_max：表示接收套接字缓冲区大小的最大值（以字节为单位）​。
  - wmem_max：表示发送套接字缓冲区大小的最大值（以字节为单位）​。
  - 建议设置16777216
- `/proc/sys/net/ipv4/tcp_rmem、tcp_wmem`
  - 可以提高Linux内核自动对Socket缓冲区进行优化的能力
  - tcp_rmem：用来配置读缓冲的大小，第1个值为最小值，第2个值为默认值，第3个值为最大值
  - tcp_wmem：用来配置写缓冲的大小，第1个值为最小值，第2个值为默认值，第3个值为最大值
  - 如：`echo "4096 87380 16777216" > /proc/sys/net/ipv4/tcp_rmem`
- `/proc/sys/net/core/somaxconn`
  - 参数用来设置Socket监听（listen）的backlog上限
  - backlog就是Socket的监听队列，当一个请求（request）尚未被处理或建立时，它会进入backlog。而Socket server可以一次性处理backlog中的所有请求，处理后的请求不再位于监听队列中。当Server处理请求较慢，以至于监听队列被填满后，新来的请求会被拒绝。
  - 默认值为128, 建议4096

### 系统Kernel参数优化

- `/proc/sys/kernel/panic`
  - 设置如果发生内核严重错误，则内核在重新引导之前等待的时间，单位s
  - 默认为0， 表示在发生内核严重错误时禁止重新引导
  - 建议为1，也就是内核故障后1s自动重启
- `/proc/sys/kernel/pid_max`
  - 参数用来设置Linux下进程数量的最大值。
  - 默认值是32768，正常情况下是够用的，当任务重时，会不够用，最终导致内存无法分配的错误
- `/proc/sys/kernel/ctrl-alt-del`
  - /proc/sys/kernel/ctrl-alt-del文件有一个二进制值，该值控制系统在接收到〈Ctrl+Alt+Delete〉组合键时如何反应
  - 0值，表示捕获〈Ctrl+Alt+Delete〉，并将其送至init程序。这将允许系统可以安全地关闭和重启，就好像输入shutdown命令一样
  - 1值，表示不捕获〈Ctrl+Alt+Delete〉
- `/proc/sys/kernel/core_pattern`
  - 参数用来设置core文件保存位置或文件名，只有文件名时，则保存在应用程序运行的目录下
  - `echo "core.%e.%p" > /proc/sys/kernel/core_pattern`
  - %e表示程序名，%p表示进程id。

### 内存内核参数

### 文件系统内核参数

优化参考配置

```ini
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_keepalive_time = 120
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_max_tw_buckets = 80000
net.ipv4.tcp_keepalive_time = 120
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_keepalive_probes = 5

net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2

net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 10

net.ipv4.tcp_max_syn_backlog = 20000
net.core.netdev_max_backlog = 32768
net.core.somaxconn = 32768

net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 2

net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_max_orphans = 3276800

net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_max_tw_buckets = 5000000
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_keepalive_probes = 5
net.nf_conntrack_max = 2097152
```

#### 回收Cache

```bash
# 释放前需要使用sync将未写的系统缓冲区写到磁盘中

# 回收page cache
echo 1 > /proc/sys/vm/drop_caches
# 释放文件节点inodes缓存和目录项缓存
echo 2 > /proc/sys/vm/drop_caches
# 释放Page Cache、文件节点inodes缓存和目录项缓存
echo 3 > /proc/sys/vm/drop_caches
```


```bash
# 统计所有用户创建的进程数

ps h -Led -o user | sort | uniq -c | sort -rn
```
