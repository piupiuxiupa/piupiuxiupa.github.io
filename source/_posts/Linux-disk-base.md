---
title: Linux 磁盘基础
date: 2024-07-17 10:53:47
tags: linux
categories: Posts
---

# Linux 磁盘基础

先从如何使用开始

- 如何查磁盘占用大小及基本信息

  将会用到的命令：

  - df
  - lsblk
  - du
  - blkid

- 如何进行磁盘分区、制作逻辑卷、格式化、挂载

  将会用到的命令：

  - fdisk
  - gdisk
  - parted
  - pvcreate
  - vgcreate
  - lvcreate
  - mkfs
  - mount

然后简单了解磁盘分区的原理

- MBR
- GPT

最后是操作磁盘时可能遇到的各种问题。例如机器添加磁盘后系统层面没有显示、已分区没做逻辑卷的磁盘扩容、曾经使用过的磁盘换到新机器使用报错、已删除文件`df -h`仍然显示占用、挂载类似nfs等网络盘时操作卡顿、实际占用与显示不符等等...



## 磁盘信息查看

### 1. 查看磁盘使用量 -- display file system

   ```bash
df -h # 二进制计算方式，以人类友好形式显示，如1024M=1G
df -l # 只列出本地文件系统，即排除nfs或其他类似的如nas盘存储挂载
df -T # 显示挂载类型
df -H # 十进制计算方式，以人类友好形式显示，如1000M=1G
df -t xfs # 指定显示xfs类型的挂载
df -i
   ```



### 2. 查看磁盘分区挂载信息 -- list block

   ```bash
lsblk
lsblk -l 
   ```

   使用`lsblk`输出如下:

   ```
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda       8:0    0   15G  0 disk
├─sda1    8:1    0   14G  0 part /
├─sda14   8:14   0    4M  0 part
├─sda15   8:15   0  106M  0 part /boot/efi
└─sda16 259:0    0  913M  0 part /boot
sr0      11:0    1   54K  0 rom
   ```

   


### 3. 查看具体文件或目录大小 -- disk usage

   ```bash
du -sh # 计算总大小
du -ahd 0 # 与上述相同
du -sh * # 计算当前目录下所有文件具体大小，最后计算出总值，并以人类友好形式输出
du -ahd 1 # 与上述相同
# -d 在有某些版本不可用，需要换成--max-depth 1
du -ahd 1 --time # 显示文件或目录修改时间
du -ahd 1 --time | sort -h ## 以人类友好形式进行排序输出
du -ahd 1 ./ -x ./ # 跳过不同文件系统的目录，经常可以用来规避nfs这类网络挂载
   ```



### 4. 其他命令

   ```bash
blkid ## block id, 可以用来查看uuid及文件系统类型
## 以下命令可以用来查看分区情况
fdisk -l 
gdisk -l
parted /dev/vda print
   ```



   ## 磁盘分区、格式化、挂载

> 磁盘有价，数据无价，对磁盘进行分区、扩容操作时请务必做好数据备份！！！

### 1. 分区

一般分区有两种常用形式，`GPT`和`MBR`

MBR格式最多只能有四个主分区，若想继续分区就需要建立逻辑分区。

GPT格式没有分区限制，建议使用GPT分区

1. **fdisk**

fdisk使用方法如下，进入交互式命令行后，用`p`查看磁盘信息，`n`新建分区，下面的操作无特殊指定回车即可。

如有指定，一般起始扇区默认即可，结束扇区可用`+10G`类似写法指定，`+`号是在现有基础上增加空间。

有时候缩容操作（一般不考虑缩容）不会指定+号，直接指定大小而不是在原基础上增加。

```bash
root@docker:~# fdisk /dev/sda

Welcome to fdisk (util-linux 2.39.3).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition number (2-13,17-128, default 2):
First sector (33556480-52428766, default 33556480):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (33556480-52428766, default 52426751):

Created a new partition 2 of type 'Linux filesystem' and of size 9 GiB.

Command (m for help): p
Disk /dev/sda: 25 GiB, 26843545600 bytes, 52428800 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: A102F557-254C-4E8A-BB7B-5CE2CBC42B40

Device        Start      End  Sectors  Size Type
/dev/sda1   2099200 33556479 31457280   15G Linux filesystem
/dev/sda2  33556480 52426751 18870272    9G Linux filesystem
/dev/sda14     2048    10239     8192    4M BIOS boot
/dev/sda15    10240   227327   217088  106M EFI System
/dev/sda16   227328  2097152  1869825  913M Linux extended boot

Partition table entries are not in disk order.

Command (m for help): w
The partition table has been altered.
Syncing disks.
```



2. **gdisk**

gdisk的操作类似于fdisk，多了一步需要输入Hex code，可回车默认。

如果保存退出后`lsblk`查看不到分区，可使用`partprobe`刷新分区表

```bash
root@docker:~# gdisk /dev/sda
GPT fdisk (gdisk) version 1.0.10

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

Command (? for help): p
Disk /dev/sda: 52428800 sectors, 25.0 GiB
Model: VBOX HARDDISK
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): A102F557-254C-4E8A-BB7B-5CE2CBC42B40
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 52428766
Partitions will be aligned on 2048-sector boundaries
Total free space is 18876348 sectors (9.0 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1         2099200        33556479   15.0 GiB    8300
  14            2048           10239   4.0 MiB     EF02
  15           10240          227327   106.0 MiB   EF00
  16          227328         2097152   913.0 MiB   EA00

Command (? for help): n
Partition number (2-128, default 2):
First sector (34-52428766, default = 33556480) or {+-}size{KMGTP}:
Last sector (33556480-52428766, default = 52426751) or {+-}size{KMGTP}: +8G
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300): 8e00
Changed type of partition to 'Linux LVM'

Command (? for help): p
Disk /dev/sda: 52428800 sectors, 25.0 GiB
Model: VBOX HARDDISK
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): A102F557-254C-4E8A-BB7B-5CE2CBC42B40
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 52428766
Partitions will be aligned on 2048-sector boundaries
Total free space is 2099132 sectors (1025.0 MiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1         2099200        33556479   15.0 GiB    8300
   2        33556480        50333695   8.0 GiB     8E00  Linux LVM
  14            2048           10239   4.0 MiB     EF02
  15           10240          227327   106.0 MiB   EF00
  16          227328         2097152   913.0 MiB   EA00

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/sda.
Warning: The kernel is still using the old partition table.
The new table will be used at the next reboot or after you
run partprobe(8) or kpartx(8)
The operation has completed successfully.

root@docker:~# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0     7:0    0    4K  1 loop /snap/bare/5
loop1     7:1    0 74.2M  1 loop /snap/core22/1380
loop2     7:2    0    1M  1 loop /snap/multipass-sshfs/145
loop3     7:3    0 38.8M  1 loop /snap/snapd/21759
sda       8:0    0   25G  0 disk
├─sda1    8:1    0   15G  0 part /
├─sda14   8:14   0    4M  0 part
├─sda15   8:15   0  106M  0 part /boot/efi
└─sda16 259:0    0  913M  0 part /boot
sr0      11:0    1   54K  0 rom
root@docker:~# partprobe -s
```



3. **parted**

> 需谨慎使用，该命令操作会直接生效，不像gdisk、fdisk先将数据写入内存

`parted` 命令也可以直接运行进入类似`fdisk`的交互式界面，但本人更习惯直接命令行操作

分区三步法：
   1. 设置分区形式 - mklabel
   2. 设置分区大小 - mkpart
      - mkpart的开始分区从第2048个扇区开始，也就是1024kb,给头部信息留足空间
   3. 按需格式化磁盘或做成pv卷扩容

```bash
## 指定分区形式以及分区名和大小
parted /dev/sda mklabel gpt mkpart primary 1024kb 100%
## 更倾向于用扇区来指定大小，假设上一个分区的结束扇区为227328，则新分区可以+1从227329扇区开始。
## 此处会自动调整结束扇区，因为磁盘尾还需要存放GPT备份表
parted /dev/sda mklabel gpt mkpart primary 227329s 100%
```

### 2. 格式化

一个磁盘分区完后还不能直接挂载使用，还需要进行文件系统格式化，格式化完成后才能挂载使用。

一般常见的为xfs、ext4格式，建议使用xfs格式，对应的命令为

```bash
mkfs.xfs /dev/sda1
mkfs.ext4 /dev/sda1

## 格式化完成后可以用blkid查看信息
blkid /dev/sda1 
```

### 3. 挂载

挂载可以是临时挂载，也可以开机自动挂载

- 临时挂载

  关机重启挂载消失

  ```bash
  mount -t xfs /dev/sda2 /tmp
  ```

- 开机自动挂载

  将挂载写入`/etc/fstab`文件即可实现开机自挂载

  ```bash
  echo "/dev/sda2   /tmp        xfs   defaults     0 0" >> /etc/fstab
  ```

### 4. 制作逻辑卷

LVM允许系统将多个物理硬盘或分区组合成一个逻辑卷组，‌从而形成一个大的存储池。利用逻辑卷的特性可以实现跨盘扩容，因此在制作逻辑卷的时候最好使用性能相近的磁盘，否则将会影响逻辑卷整体性能。

```bash
## 先将某块磁盘或分区创建为PV
pvcreate /dev/sda
## 将sda创建一个名为vg1的卷组
vgcreate vg1 /dev/sda
## 在vg1卷组创建一个名为lv1的逻辑卷，将所有vg1空间都使用
lvcreate -l +100%free -n lv1 vg1
## 在vg1卷组创建一个名为lv1,大小为40G的逻辑卷
lvcreate -L 40G -n lv1 vg1
## 制作文件系统
mkfs.xfs /dev/vg1/lv1
## 挂载，同样可以写入fstab
mount /dev/vg1/lv1 /root/tmp/
```

### 5. 磁盘扩容

通常磁盘需要做逻辑卷或者处于最后一个分区的时候进行扩容是最方便的。

扩容前一定必须做好备份！！！

1. 逻辑卷扩容

```bash
## 首先加入pv
pvcreate /dev/sda
## 然后扩展vg
vgextend vg1 /dev/sda
## 扩充lv
## +号必须有，不然就变成了调整lv大小，造成数据丢失
lvextend -l +100%free /dev/vg1/lv1
lvextend -L +20G /dev/vg1/lv1
## 同步文件系统
## xfs 
xfs_growfs /dev/vg1/lv1
## ext4
resize2fs /dev/vg1/lv1
```

2. 没分区的整块盘

```bash
## 可以直接同步文件系统
xfs_growfs /dev/sda
resize2fs /dev/sda
```



## 磁盘分区类型及原理

#### MBR（Master Boot Record）分区格式

早期磁盘第一个扇区（`521bytes`）里面包含重要的信息`MBR（Master Boot Record）`，其中`446 bytes`，安装开机管理程序的地方；剩下的`64bytes`记录硬盘分区的数据，即分区表，如图

![mbr](../images/Linux-disk-base/MBR.png)

由于分区表所在区块仅有64bytes容量，因此最多仅能有四组记录区，每组记录区记录了该区段的起始与结束的磁柱号码。

也就是MBR只支持四个主分区，且最大只支持2TB的硬盘。

若想多个分区，需要使用扩展分区，扩展分区从5开始计数。

#### GPT（GUID Partition Table）分区格式

GPT将磁盘划分为一块块的`逻辑区块地址（Logical Block Address，简称LBA）` 来处理，每个LBA预设计为512bytes，即一个扇区的大小；同时改进MBR之用一块扇区来标识分区表的弊端，GPT使用了前后各34个LBA来标识分区表信息（最后的34各区可以理解为备份，达到高可用），如图

![gpt](../images/Linux-disk-base/GPT.png)

 LBA的标识是从0开始的，LBA0-34共35块，这里分别阐述下其含义:

- `LBA0` :包含两部分，一部分是类似MBR的446bytes,存储开机管理程序，第二部分则是存储一个特殊的标记，标识该磁盘为GPT格式，而看不懂GPT分区的程序则无法操作该磁盘，起到保护作用，放心，目前基本的管理程序都能识别GPT格式，所以该LBA块实际上与分区信息并无直接关联，这就是为啥不算入34LBA的原因
- `LBA1` :GPT的表头，记录分区本身的位置与大小，同时记录分区在备份中最后34个LBA中的位置，方便恢复
- `LBA2-34`:共32块LBA，每块LBA记录4笔分区表，共支持4\*32=128笔分区；而每个LBA默认为512bytes，则每笔记录用到512/4=128bytes,每笔记录拿出64bytes来记录开始、结束的扇区号码，因此对一个单一分区槽而言，支持的最大容量为2^64∗512bytes=2^63∗1Kbytes=2^33TB=8ZB

> 1024TB=1PB \
> 1024PB=1EB \
> 1024EB=1ZB



# 其他事项

## 1. 已添加磁盘无法识别

有时候机房加了磁盘可能不会在系统层面立即显示出来，需要重启服务器，也可以重新扫描下scsi总线

```bash
for i in /sys/class/scsi_host/*/scan; do echo $i;echo "- - -" > $i; done
```



## 2. 已分区没做lvm的磁盘扩容

有时候我们会遇到项目组在原磁盘上加空间，而想要扩容的那个分区是没有做逻辑卷的，以至于加的空间会浪费，下面有两种方案，其一是`fdisk`的一种特殊用法，其二是提供一个工具`growpart`，可以对没有做过逻辑卷的分区进行扩容。

两种方案都需要该分区必需是整个磁盘的最后一个分区

- 其一，使用`fdisk`扩容。因为fdisk在进行操作的时候分区数据不会直接写入磁盘，而是先会保存在内存中，所以可以利用这点来进行扩容

  ```bash
  fdisk /dev/sda
  ## 进入后用p查看分区
  ## 按d 删除最后一个分区
  ## 然后再n，创建新分区。
  ## 新分区的大小必须比原来的分区大，不能缩小，否则会造成数据丢失
  ## 最后w保存，然后xfs_growf或resize2fs同步对应文件系统即可
  ```
  
- 其二，可使用`growpart`来进行扩容

   ```bash
   # 联网条件下
   yum install cloud-utils-growpart
   # 没联网就传包吧
   rpm -ivhU cloud-utils-growpart.rpm
   # 如服务器有一块盘vda，仅有一个分区vda1，且没做过逻辑卷
   # 先用growpart将空间加到vda1上
   ## 表示对/dev/sda的分区1进行扩容
   growpart /dev/vda 1
   ## 如果报错 unexpected output in sfdisk --version
   ## LANG=en_US.UTF-8 就可以了,不行可以重启下物理机试一下.(编码问题)
   # 然后同步文件系统即可
   xfs_growfs /
   resize2fs /dev/vda1
   ```

**案例**

>由于没有找到现成的做了分区而没做lvm的项目，但是步骤是一样的。

![gdisk_caution](../images/Linux-disk-base/20221031165313.jpg)


## 3. 旧磁盘换到新机器无法使用

面对这个情况其实如果在确定磁盘里面的数据进行了备份，或者不需要的时候可以直接进行强制格式化。

也可以`dd`去备份一下磁盘头和磁盘尾信息，或者整个磁盘。

如下有一个磁盘sda，通过parted查看磁盘信息，这里建议把unit切换成s单位来看会更精确更清楚些。

```shell
root@docker:~# parted /dev/sda u s p
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sda: 31457280s
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start     End        Size       File system  Name  Flags
14      2048s     10239s     8192s                         bios_grub
15      10240s    227327s    217088s    fat32              boot, esp
16      227328s   2097152s   1869825s   ext4               bls_boot
 1      2099200s  31457246s  29358047s  ext4
```

在这个输出中可以看到磁盘总大小31457280个扇区，磁盘实际分配到第31457246个扇区结束，`31457280-31457246=34`，也就是刚好留下了GPT格式备份的34个LBA。

```bash
## dd默认1count为512B，与扇区size对应
## 备份前34个扇区
dd if=/dev/sda of=~/sda.img.bak count=34
## 跳过前面31457246个扇区进行备份
dd if=/dev/sda of=~/sda.img.bak skip=31457246
```

如下输出可见备份了最后34个扇区


```shell
root@docker:~# dd if=/dev/sda of=~/sda.end.bak skip=31457246
34+0 records in
34+0 records out
17408 bytes (17 kB, 17 KiB) copied, 0.000214267 s, 81.2 MB/s
```

备份好后可以直接用`dd`命令把磁头磁尾清空即可

```bash
dd if=/dev/zero of=/dev/sda count=34
dd if=/dev/zero of=/dev/sda skip=31457246
```



## 4. 已删除文件占用空间

这种情况一般是有进程在占用被删除的文件，从而导致空间未释放

```bash
## 可使用lsof来排查，找到对应的pid
lsof | grep deleted
## 非业务时间段可以通过 kill对应进程来释放
## 但业务时间段肯定不能这样操作
## 所以有了第二种方法
ls -l /proc/$pid/fd/* | grep $filename
echo > /proc/$pid/fd/$fdnum
```

## 5. 实际占用与显示不符

一可能是前面说的已删除文件后`df`没有变化

二可能是覆盖挂载

三可能是计算的时候没有排除网络挂载

前面已经介绍过第一种处理办法，下面介绍后面两种

对于覆盖挂载可以在非业务时间将异常挂载的目录先卸载下来，然后去看看卸载后的该目录下面是否有文件，如有文件占用空间则属于覆盖挂载。

对于网络挂载，可以在使用`du`命令的时候使用`-x`选项来进行排除（前面有介绍过），便可计算出当前本地的准确值。

## 6. 由MBR (MSDOS) 格式转GPT
**场景**

MBR分区只支持四个主分区，因此，当有人在第四个分区不用逻辑分区而继续用主分区的话将无法再进行分区，且最大只能操作2T空间的磁盘，所以在面对这种四个主分区都用满或磁盘空间超过2T的情况下，需要将磁盘格式转成GPT后才能进一步进行分区和扩容。

**前提条件**

磁盘头信息需要留足空间转换成GPT分区，start最小需要从第63个扇区开始。

可以自己模拟下，用parted工具进行分区，第一个分区头从0开始，命令如下。

```bash
parted /dev/vdx mklabel msdos/gpt mkpart primary 0 100%
```

如果使用的label是mbr则会默认从63s开始，如果是gpt，则会从34s开始。

![mbr_default](../images/Linux-disk-base/078092image.png)

![gpt_default](../images/Linux-disk-base/098712image.png)

如果用的是gdisk和fdisk分区，默认是从2048s开始，所以完全够从mbr转换成gpt格式。

但是需要注意磁盘末尾是否还有空间，因为MBR末尾并不记录信息，所以当磁盘100%时也就是空间完全没有了，此时进行gpt转换可能造成尾部数据丢失。

![fdisk_default](../images/Linux-disk-base/052384image.png)

- 先备份分区表信息

```bash
# 备份分区表信息
# 最好选择另外的盘进行备份
dd if=/dev/vda of=/xxxx/xxx.mbr.bak  bs=1024 count=2000
# 恢复分区表信息
dd if=/home/xxx.mbr.bak  of=/dev/vda bs=1024 count=2000
```

- 使用gdisk将分区格式转换为gpt

![gdisk_default](../images/Linux-disk-base/016511image.png)


## 7. 新磁盘残留GPT分区信息导致无法分区扩容

> 类似第三点

当项目组拿来新磁盘，要求分区扩容时，碰到如下提示gpt备份区信息与主分区信息不一致的提示时，考虑是该磁盘是曾经做过分区的盘。

![gdisk_caution](../images/Linux-disk-base/20221029005856.png)

先与项目组确认是否是新盘无数据，当确认是新盘无数据后，可用`dd`命令去清空头尾记录的分区信息，一般取首尾1M左右即可，清空完后就可以按正常流程进行分区。

```bash
# 如项目组有893G磁盘
# 清空头部信息
dd if=/dev/zero of=/dev/sdb bs=512 count=4000
# 清空尾部信息
dd if=/dev/zero of=/dev/sdb bs=1M seek=890000
# seek参数为跳过sdb的第890000个block，即将if的内容从第890000个block后开始写入
```