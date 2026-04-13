---
title: QEMU 完整使用教程
date: 2026-04-13 10:21:56
tags: [tools, linux]
categories: Posts
---

## 目录

1. [安装与基础概念](#1-安装与基础概念)
2. [磁盘镜像管理](#2-磁盘镜像管理qemu-img)
3. [创建并启动第一个 VM](#3-创建并启动第一个-vm)
4. [网络配置](#4-网络配置)
5. [文件共享与挂载](#5-文件共享与挂载)
6. [USB 与设备直通](#6-usb-与设备直通)
7. [快照与备份](#7-快照与备份)
8. [性能优化](#8-性能优化)
9. [无头模式与远程管理](#9-无头模式与远程管理)
10. [macOS 特殊配置](#10-macos-特殊配置)
11. [实用脚本模板](#11-实用脚本模板)
12. [常见问题排查](#12-常见问题排查)

---

## 1. 安装与基础概念

### 安装

```bash
brew install qemu

# 验证
qemu-system-aarch64 --version   # Apple Silicon
qemu-system-x86_64 --version    # Intel Mac（也能在 ARM Mac 上跑）
qemu-img --version
```

### 核心概念

QEMU 有两种运行模式：

| 模式 | 说明 | 命令 |
|---|---|---|
| **系统模拟** | 完整的虚拟机，有自己的内核 | `qemu-system-x86_64` / `qemu-system-aarch64` |
| **用户态模拟** | 在宿主上直接运行为其他架构编译的二进制 | `qemu-aarch64` / `qemu-x86_64` |

本教程聚焦 **系统模拟** 模式。

### macOS 上的加速方案

| 加速器 | 适用 | 命令参数 | 说明 |
|---|---|---|---|
| **HVF** | macOS 专用 | `-accel hvf` | Hypervisor.framework，Apple 原生，推荐 |
| **TCG** | 通用 | `-accel tcg` | 纯软件模拟，慢但兼容性好 |
| **TCG + 多线程** | 通用 | `-accel tcg,thread=multi` | 多线程软件模拟 |

> **Apple Silicon Mac 只能用 HVF 跑 ARM 系统，不能用 HVF 跑 x86 系统。** 跑 x86 会 fallback 到 TCG（软件模拟），非常慢。

### QEMU 命令的基本结构

```bash
qemu-system-<架构> \
  -accel <加速器> \          # 加速方式
  -m <内存> \                # 内存大小
  -smp <核数> \              # CPU 核心数
  -cpu <CPU型号> \           # CPU 类型
  -drive file=<镜像>,... \   # 磁盘
  -netdev <网络后端>,... \    # 网络后端
  -device <网络前端>,... \    # 网络设备
  -bios <固件> \             # UEFI/BIOS 固件
  [其他选项]
```

---

## 2. 磁盘镜像管理（qemu-img）

### 支持的格式

| 格式 | 扩展名 | 特点 |
|---|---|---|
| **qcow2** | `.qcow2` | QEMU 原生格式，支持快照、压缩、加密、稀疏，**推荐** |
| **raw** | `.img` / `.raw` | 原始格式，性能最好，但不支持快照 |
| **vmdk** | `.vmdk` | VMware 格式，用于兼容 |
| **vdi** | `.vdi` | VirtualBox 格式 |

### 常用操作

```bash
# 创建 qcow2 镜像（推荐）
qemu-img create -f qcow2 myvm.qcow2 40G
# 实际只占几百 KB，随用随增长（稀疏）

# 创建 raw 镜像
qemu-img create -f raw myvm.raw 40G

# 查看镜像信息
qemu-img info myvm.qcow2

# 调整大小（只能扩大，不能缩小 qcow2）
qemu-img resize myvm.qcow2 60G

# 格式转换
qemu-img convert -f raw -O qcow2 input.img output.qcow2
qemu-img convert -f vmdk -O qcow2 vmware.vmdk qemu.qcow2
# raw → qcow2（加压缩）
qemu-img convert -f raw -O qcow2 -c input.raw output.qcow2

# 验证镜像完整性
qemu-img check myvm.qcow2
```

### 镜像类型

```bash
# 动态分配（默认，推荐）
qemu-img create -f qcow2 dynamic.qcow2 100G   # 初始只占 ~300KB

# 预分配全部分配（性能更好）
qemu-img create -f qcow2 -o preallocation=full preallocated.qcow2 40G

# 加密
qemu-img create -f qcow2 -o encrypt.format=luks -o encrypt.key-secret=secret0 encrypted.qcow2 40G
```

### Backing File（增量镜像）

```bash
# 基础镜像（只读）
qemu-img create -f qcow2 base.qcow2 40G
# 在基础镜像上安装好系统...

# 创建增量镜像（基于 base，写入到 overlay）
qemu-img create -f qcow2 -b base.qcow2 -F qcow2 overlay.qcow2

# 启动 overlay（修改不会影响 base）
qemu-system-aarch64 ... -drive file=overlay.qcow2

# 可以从同一个 base 创建多个 overlay，互不影响
qemu-img create -f qcow2 -b base.qcow2 -F qcow2 test1.qcow2
qemu-img create -f qcow2 -b base.qcow2 -F qcow2 test2.qcow2
```

应用场景：基础镜像装好系统后，为不同测试场景创建多个增量镜像，互不干扰。

---

## 3. 创建并启动第一个 VM

### 场景 A：Apple Silicon — ARM64 Linux

#### 下载资源

```bash
mkdir -p ~/qemu-vms/arm-linux && cd ~/qemu-vms/arm-linux

# 下载 Ubuntu 22.04 ARM64 云镜像
wget https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-arm64.img

# 下载 UEFI 固件
# Homebrew 安装的 QEMU 自带
ls /opt/homebrew/share/qemu/edk2-aarch64-code.fd
# 如果没有：
# wget https://releases.linaro.org/components/kernel/uefi-linaro/latest/release/qemu64/QEMU_EFI.fd
```

#### 准备 cloud-init

```bash
# 获取你的 SSH 公钥
cat ~/.ssh/id_ed25519.pub
# 如果没有：ssh-keygen -t ed25519
```

创建 `meta-data`：

```yaml
instance-id: arm-linux-01
local-hostname: arm-linux
```

创建 `user-data`：

```yaml
#cloud-config
hostname: arm-linux
manage_etc_hosts: true

users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAA...YOUR_PUBLIC_KEY_HERE...

packages:
  - vim
  - curl
  - git

runcmd:
  - systemctl enable ssh
```

生成 seed 盘：

```bash
brew install cloud-localds
cloud-localds seed.img user-data meta-data
```

#### 启动 VM

```bash
# 扩容镜像
qemu-img resize ubuntu-22.04-server-cloudimg-arm64.img 20G

# 启动
qemu-system-aarch64 \
  -machine virt,accel=hvf \
  -cpu cortex-a72 \
  -smp 4 \
  -m 4G \
  -bios /opt/homebrew/share/qemu/edk2-aarch64-code.fd \
  -drive if=virtio,file=ubuntu-22.04-server-cloudimg-arm64.img,format=qcow2 \
  -drive if=virtio,file=seed.img,format=raw \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net0 \
  -nographic
```

参数逐个解释：

| 参数 | 含义 |
|---|---|
| `-machine virt` | 使用 QEMU 的 `virt` 虚拟机类型（ARM 推荐） |
| `,accel=hvf` | 使用 macOS Hypervisor 加速 |
| `-cpu cortex-a72` | 模拟 ARM Cortex-A72 CPU |
| `-smp 4` | 4 个 vCPU |
| `-m 4G` | 4 GB 内存 |
| `-bios ...` | UEFI 固件路径 |
| `-drive if=virtio,...` | 使用 virtio 驱动的磁盘（性能好） |
| `-netdev user,id=net0,...` | 用户态网络，端口转发 2222→22 |
| `-device virtio-net-pci,...` | 虚拟网卡 |
| `-nographic` | 无图形界面，终端直接看输出 |

#### 连接

```bash
# 等启动完成后（看到 login 提示）
ssh -p 2222 ubuntu@localhost
```

### 场景 B：Intel Mac — x86_64 Linux

```bash
mkdir -p ~/qemu-vms/x86-linux && cd ~/qemu-vms/x86-linux

# 下载 Ubuntu 22.04 x86_64 云镜像
wget https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-amd64.img

# 扩容
qemu-img resize ubuntu-22.04-server-cloudimg-amd64.img 20G

# cloud-init 同上...

# 启动（注意：Intel Mac 用 hvf 加速 x86）
qemu-system-x86_64 \
  -machine accel=hvf \
  -smp 4 \
  -m 4G \
  -drive if=virtio,file=ubuntu-22.04-server-cloudimg-amd64.img,format=qcow2 \
  -drive if=virtio,file=seed.img,format=raw \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net0 \
  -nographic

ssh -p 2222 ubuntu@localhost
```

### 场景 C：从 ISO 全新安装

```bash
# 下载 ISO
wget https://releases.ubuntu.com/22.04/ubuntu-22.04.4-live-server-amd64.iso

# 创建空磁盘
qemu-img create -f qcow2 ubuntu-install.qcow2 40G

# 首次启动：从 ISO 引导（Apple Silicon 示例）
qemu-system-aarch64 \
  -machine virt,accel=hvf \
  -cpu cortex-a72 \
  -smp 4 \
  -m 4G \
  -bios /opt/homebrew/share/qemu/edk2-aarch64-code.fd \
  -drive if=virtio,file=ubuntu-install.qcow2,format=qcow2 \
  -cdrom ubuntu-22.04.4-live-server-arm64.iso \
  -boot d \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net0 \
  -nographic

# 安装完成后，去掉 -cdrom 和 -boot d，从磁盘启动
qemu-system-aarch64 \
  -machine virt,accel=hvf \
  -cpu cortex-a72 \
  -smp 4 \
  -m 4G \
  -bios /opt/homebrew/share/qemu/edk2-aarch64-code.fd \
  -drive if=virtio,file=ubuntu-install.qcow2,format=qcow2 \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net0 \
  -nographic
```

---

## 4. 网络配置

### 4.1 User 模式（默认，最简单）

VM 通过 QEMU 内置的 NAT/DHCP 访问外网。**宿主机 → VM** 需要端口映射。

```bash
-netdev user,id=net0,hostfwd=tcp::2222-:22
```

完整端口映射语法：

```bash
-netdev user,id=net0,\
  hostfwd=tcp::2222-:22,\      # SSH
  hostfwd=tcp::8080-:80,\      # HTTP
  hostfwd=tcp::4433-:443,\     # HTTPS
  hostfwd=tcp::5432-:5432,\    # PostgreSQL
  hostfwd=tcp::6379-:6379      # Redis
```

| 特性 | 支持情况 |
|---|---|
| VM → 外网 | ✅ |
| 宿主机 → VM | ✅ 需端口映射 |
| 其他设备 → VM | ❌ |
| VM ← 宿主机 IP | 10.0.2.2（默认网关） |

### 4.2 Socket VMNet（推荐进阶方案）

让 VM 获得独立 IP，局域网其他设备也能访问。

```bash
# 安装 socket_vmnet
brew install socket_vmnet

# 创建 socket_vmnet 服务配置
sudo cat > /Library/LaunchDaemons/com.github.socket_vmnet.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.github.socket_vmnet</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/opt/socket_vmnet/bin/socket_vmnet</string>
    <string>--vmnet-mode=shared</string>
    <string>--socket-path=/tmp/socket_vmnet</string>
    <string>--vmnet-gateway=192.168.106.1</string>
    <string>--vmnet-dhcp-end=192.168.106.254</string>
    <string>en0</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>UserName</key>
  <string>root</string>
</dict>
</plist>
EOF

# 启动服务
sudo launchctl load /Library/LaunchDaemons/com.github.socket_vmnet.plist
```

在 VM 启动命令中替换网络部分：

```bash
# 网络通过 fd 传递
-netdev socket,id=net0,fd=3 \
-device virtio-net-pci,netdev=net0
```

或者更简单地用 `vmnet.framework` 直接桥接：

```bash
# macOS 原生 vmnet 桥接（需要 root）
-netdev vmnet-shared,id=net0 \
-device virtio-net-pci,netdev=net0
```

### 4.3 端口转发速查表

| 服务 | VM 端口 | 映射示例 |
|---|---|---|
| SSH | 22 | `hostfwd=tcp::2222-:22` |
| HTTP | 80 | `hostfwd=tcp::8080-:80` |
| HTTPS | 443 | `hostfwd=tcp::4433-:443` |
| MySQL | 3306 | `hostfwd=tcp::3306-:3306` |
| PostgreSQL | 5432 | `hostfwd=tcp::5432-:5432` |
| Redis | 6379 | `hostfwd=tcp::6379-:6379` |
| MongoDB | 27017 | `hostfwd=tcp::27017-:27017` |

---

## 5. 文件共享与挂载

### 5.1 SCP/SFTP（最通用）

```bash
# 宿主机 → VM
scp -P 2222 myfile.txt ubuntu@localhost:~

# VM → 宿主机
scp -P 2222 ubuntu@localhost:/path/to/file ./

# 用 rsync
rsync -avz -e "ssh -p 2222" ./local-dir/ ubuntu@localhost:~/remote-dir/
```

### 5.2 9p 文件系统共享（QEMU 原生）

在启动命令中添加：

```bash
# 共享宿主机的 ~/projects 目录到 VM 的 /shared
-virtfs local,path=$HOME/projects,mount_tag=hostshare,security_model=mapped-xattr,id=share0
```

在 VM 内挂载：

```bash
sudo mkdir -p /mnt/host
sudo mount -t 9p -o trans=virtio,version=9p2000.L hostshare /mnt/host

# 开机自动挂载（加到 /etc/fstab）
echo 'hostshare /mnt/host 9p trans=virtio,version=9p2000.L,_netdev 0 0' | sudo tee -a /etc/fstab
```

### 5.3 通过 SSHFS（从 VM 侧挂载宿主机）

在 VM 内：

```bash
sudo apt install sshfs
mkdir -p ~/mac
sshfs -p 2222 ubuntu@localhost:/Users/yourname ~/mac
```

### 5.4 方案对比

| 方案 | 性能 | 配置难度 | 双向同步 |
|---|---|---|---|
| SCP/SFTP | 一般 | 零 | ❌ 手动 |
| 9p virtfs | 较好 | 中 | ✅ 实时 |
| SSHFS | 一般 | 低 | ✅ 实时 |

---

## 6. USB 与设备直通

### 查看宿主机 USB 设备

```bash
# 列出 USB 设备
system_profiler SPUSBDataType
```

### 传递 USB 设备

```bash
# 启动时添加 USB 控制器
-device qemu-xhci,id=xhci \     # USB 3.0 控制器
-device usb-host,vendorid=0x1234,productid=0x5678  # 指定设备
```

或者动态挂载（需要 QEMU Monitor）：

```
# 在 QEMU Monitor 中（Ctrl+A → C 切换到 monitor）
(qemu) device_add usb-host,vendorid=0x1234,productid=0x5678
(qemu) device_del <device-id>   # 移除
```

### 完整示例：传递 USB 闪存盘

```bash
# 1. 查看 USB 设备的 vendor:product ID
system_profiler SPUSBDataType | grep -A5 "Flash Drive"
# 假设得到 vendorid=0x0781, productid=0x5581

# 2. 启动 VM（加上图形界面和 USB 支持）
qemu-system-aarch64 \
  -machine virt,accel=hvf \
  -cpu cortex-a72 \
  -smp 4 -m 4G \
  -bios /opt/homebrew/share/qemu/edk2-aarch64-code.fd \
  -drive if=virtio,file=myvm.qcow2 \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net0 \
  -device qemu-xhci,id=xhci \
  -device usb-host,vendorid=0x0781,productid=0x5581 \
  -display cocoa   # macOS 图形窗口
```

---

## 7. 快照与备份

### QEMU Monitor 内的快照

QEMU Monitor 是运行中的 VM 的管理控制台：

- `-nographic` 模式：按 `Ctrl+A`，再按 `C` 进入
- 图形模式：菜单栏 → View → compatmonitor0

```bash
# 创建快照
(qemu) savevm my-snapshot-name

# 列出快照
(qemu) info snapshots

# 恢复快照
(qemu) loadvm my-snapshot-name

# 删除快照
(qemu) delvm my-snapshot-name
```

### 用 qemu-img 管理快照

```bash
# 列出镜像中的快照
qemu-img snapshot -l myvm.qcow2

# 创建快照（需要 VM 关机状态）
qemu-img snapshot -c snapshot-name myvm.qcow2

# 恢复快照
qemu-img snapshot -a snapshot-name myvm.qcow2

# 删除快照
qemu-img snapshot -d snapshot-name myvm.qcow2
```

### 完整备份流程

```bash
# 方法 1：复制镜像文件（需关机）
cp myvm.qcow2 myvm-backup-$(date +%Y%m%d).qcow2

# 方法 2：导出为压缩文件（需关机）
qemu-img convert -f qcow2 -O qcow2 -c myvm.qcow2 backup.qcow2

# 方法 3：使用增量镜像做时间点备份
qemu-img create -f qcow2 -b myvm.qcow2 -F qcow2 backup-$(date +%Y%m%d).qcow2
```

---

## 8. 性能优化

### CPU

```bash
# Apple Silicon：固定到 host CPU 类型
-cpu max                    # 使用最大特性集
-cpu cortex-a72             # 稳定的 ARM Cortex-A72

# 多核配置
-smp 4                      # 4 核
-smp 4,sockets=1,cores=4,threads=1   # 详细拓扑
```

### 磁盘 I/O

```bash
# 使用 virtio 驱动（比 IDE/SCSI 快得多）
-drive if=virtio,file=disk.qcow2,format=qcow2

# 使用 iothread（减少主线程阻塞）
-object iothread,id=io1 \
-device virtio-blk-pci,drive=drive0,iothread=io1 \
-drive id=drive0,file=disk.qcow2,format=qcow2,if=none

# 写回缓存（性能优先，断电可能丢数据）
-drive if=virtio,file=disk.qcow2,format=qcow2,cache=writeback

# 禁用磁盘缓存（数据安全优先）
-drive if=virtio,file=disk.qcow2,format=qcow2,cache=none
```

### 网络

```bash
# virtio 网卡（比 e1000 快得多）
-device virtio-net-pci,netdev=net0

# 多队列网卡
-device virtio-net-pci,netdev=net0,mq=on,vectors=6
```

### 内存

```bash
# 大页内存（减少 TLB miss）
-m 4G -mem-path /tmp/hugepages
```

### 缓存策略对比

| cache 模式 | 性能 | 安全性 | 适用场景 |
|---|---|---|---|
| `none` | 低 | 高 | 数据库等安全敏感 |
| `writeback` | 高 | 低 | 开发测试 |
| `writethrough` | 中 | 中 | 通用 |
| `unsafe` | 最高 | 最低 | 安装系统时临时用 |

### 推荐的生产级配置模板

```bash
qemu-system-aarch64 \
  -machine virt,accel=hvf \
  -cpu max \
  -smp 4 \
  -m 8G \
  -bios /opt/homebrew/share/qemu/edk2-aarch64-code.fd \
  \
  `# 磁盘：virtio + iothread + writeback` \
  -object iothread,id=io1 \
  -device virtio-blk-pci,drive=sysdisk,iothread=io1 \
  -drive id=sysdisk,file=myvm.qcow2,format=qcow2,if=none,cache=writeback,discard=unmap \
  \
  `# 网络：virtio` \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net0 \
  \
  `# 串口控制台` \
  -serial mon:stdio \
  \
  `# 优雅关机支持` \
  -no-reboot \
  -nographic
```

---

## 9. 无头模式与远程管理

### 后台运行（daemonize）

```bash
qemu-system-aarch64 \
  ... \
  -nographic \
  -serial mon:stdio \
  -daemonize \
  -pidfile /tmp/qemu-myvm.pid

# 查看进程
cat /tmp/qemu-myvm.pid

# 停止 VM
kill $(cat /tmp/qemu-myvm.pid)
```

### QEMU Monitor via Socket

```bash
# 启动时添加 monitor socket
qemu-system-aarch64 \
  ... \
  -monitor unix:/tmp/qemu-monitor.sock,server,nowait

# 从另一个终端连接 monitor
socat - UNIX-CONNECT:/tmp/qemu-monitor.sock
# 或者
nc -U /tmp/qemu-monitor.sock

# 通过 monitor 执行命令
(qemu) info status
(qemu) info blockstats
(qemu) system_powerdown    # 优雅关机
(qemu) quit                 # 强制退出
```

### QMP（QEMU Machine Protocol）

JSON 格式的管理协议，适合编程控制：

```bash
# 启动时启用 QMP
qemu-system-aarch64 \
  ... \
  -qmp unix:/tmp/qemu-qmp.sock,server,nowait

# 连接并发送命令
echo '{"execute":"qmp_capabilities"}' | socat - UNIX-CONNECT:/tmp/qemu-qmp.sock
echo '{"execute":"query-status"}' | socat - UNIX-CONNECT:/tmp/qemu-qmp.sock
```

### VNC 远程图形访问

```bash
# 启动时启用 VNC
qemu-system-aarch64 \
  ... \
  -vnc :0   # 监听 5900 端口，无密码

# 带密码
qemu-system-aarch64 \
  ... \
  -vnc :0,password=on
# 在 monitor 中设置密码
# (qemu) change vnc password mypassword

# 从客户端连接
# macOS: 用 Screen Sharing 或 brew install tiger-vnc
vncviewer localhost:5900
```

---

## 10. macOS 特殊配置

### 显示模式

```bash
# 无图形界面（服务器场景）
-nographic

# macOS 原生窗口
-display cocoa

# VNC
-vnc :0

# 无显示（后台运行）
-display none
```

### Serial Console 优化

```bash
# 将串口输出到终端，同时保留 monitor
-serial mon:stdio

# 这允许你：
# - 直接在终端看到 VM 启动日志
# - Ctrl+A, C 切换到 QEMU Monitor
# - Ctrl+A, X 直接退出 QEMU
```

### UEFI 固件位置

```bash
# Homebrew 安装的 QEMU 固件路径
# ARM64
/opt/homebrew/share/qemu/edk2-aarch64-code.fd

# x86_64
/opt/homebrew/share/qemu/edk2-x86_64-code.fd
# 或者
/usr/share/qemu/OVMF.fd      # Linux
```

---

## 11. 实用脚本模板

### 一键启动脚本

创建 `~/qemu-vms/start-vm.sh`：

```bash
#!/bin/bash
set -euo pipefail

VM_NAME="${1:-default}"
VM_DIR="$HOME/qemu-vms/$VM_NAME"
PID_FILE="$VM_DIR/qemu.pid"
DISK="$VM_DIR/disk.qcow2"
MONITOR_SOCK="$VM_DIR/qemu-monitor.sock"

# 默认配置
CPU=${QEMU_CPU:-4}
MEMORY=${QEMU_MEMORY:-4G}
DISK_SIZE=${QEMU_DISK:-40G}
SSH_PORT=${QEMU_SSH_PORT:-2222}

mkdir -p "$VM_DIR"

# 创建磁盘（如果不存在）
if [ ! -f "$DISK" ]; then
    echo "Creating new disk: $DISK ($DISK_SIZE)"
    qemu-img create -f qcow2 "$DISK" "$DISK_SIZE"
fi

# 检查是否已在运行
if [ -f "$PID_FILE" ] && kill -0 "$(cat "$PID_FILE")" 2>/dev/null; then
    echo "VM '$VM_NAME' is already running (PID: $(cat "$PID_FILE"))"
    echo "SSH: ssh -p $SSH_PORT ubuntu@localhost"
    echo "Monitor: socat - UNIX-CONNECT:$MONITOR_SOCK"
    exit 0
fi

# 清理旧 socket
rm -f "$MONITOR_SOCK"

echo "Starting VM '$VM_NAME'..."
echo "  CPU: $CPU cores"
echo "  Memory: $MEMORY"
echo "  Disk: $DISK"
echo "  SSH: ssh -p $SSH_PORT ubuntu@localhost"

qemu-system-aarch64 \
  -machine virt,accel=hvf \
  -cpu cortex-a72 \
  -smp "$CPU" \
  -m "$MEMORY" \
  -bios /opt/homebrew/share/qemu/edk2-aarch64-code.fd \
  -drive if=virtio,file="$DISK",format=qcow2,cache=writeback \
  -netdev user,id=net0,hostfwd=tcp::${SSH_PORT}-:22 \
  -device virtio-net-pci,netdev=net0 \
  -serial mon:stdio \
  -monitor unix:"$MONITOR_SOCK",server,nowait \
  -daemonize \
  -pidfile "$PID_FILE" \
  -nographic

echo "VM started. PID: $(cat "$PID_FILE")"
echo ""
echo "  SSH:      ssh -p $SSH_PORT ubuntu@localhost"
echo "  Monitor:  socat - UNIX-CONNECT:$MONITOR_SOCK"
echo "  Stop:     $0 stop $VM_NAME"
```

### 停止脚本

创建 `~/qemu-vms/stop-vm.sh`：

```bash
#!/bin/bash
set -euo pipefail

VM_NAME="${1:-default}"
VM_DIR="$HOME/qemu-vms/$VM_NAME"
PID_FILE="$VM_DIR/qemu.pid"
MONITOR_SOCK="$VM_DIR/qemu-monitor.sock"

if [ ! -f "$PID_FILE" ]; then
    echo "VM '$VM_NAME' is not running (no PID file)"
    exit 0
fi

PID=$(cat "$PID_FILE")
if ! kill -0 "$PID" 2>/dev/null; then
    echo "VM '$VM_NAME' is not running (stale PID file)"
    rm -f "$PID_FILE"
    exit 0
fi

# 尝试优雅关机
if [ -S "$MONITOR_SOCK" ]; then
    echo "Sending powerdown command..."
    echo "system_powerdown" | socat - UNIX-CONNECT:"$MONITOR_SOCK" 2>/dev/null || true
    sleep 5
fi

# 检查是否关了
if kill -0 "$PID" 2>/dev/null; then
    echo "Force killing..."
    kill "$PID"
    sleep 2
fi

rm -f "$PID_FILE" "$MONITOR_SOCK"
echo "VM '$VM_NAME' stopped."
```

### 快照管理脚本

创建 `~/qemu-vms/snapshot.sh`：

```bash
#!/bin/bash
set -euo pipefail

ACTION="${1:-list}"
VM_NAME="${2:-default}"
SNAP_NAME="${3:-}"

VM_DIR="$HOME/qemu-vms/$VM_NAME"
DISK="$VM_DIR/disk.qcow2"

case "$ACTION" in
    list)
        echo "Snapshots for '$VM_NAME':"
        qemu-img snapshot -l "$DISK"
        ;;
    create)
        [ -z "$SNAP_NAME" ] && { echo "Usage: $0 create <vm> <snapshot-name>"; exit 1; }
        qemu-img snapshot -c "$SNAP_NAME" "$DISK"
        echo "Snapshot '$SNAP_NAME' created."
        ;;
    restore)
        [ -z "$SNAP_NAME" ] && { echo "Usage: $0 restore <vm> <snapshot-name>"; exit 1; }
        qemu-img snapshot -a "$SNAP_NAME" "$DISK"
        echo "Snapshot '$SNAP_NAME' restored. Start VM to use it."
        ;;
    delete)
        [ -z "$SNAP_NAME" ] && { echo "Usage: $0 delete <vm> <snapshot-name>"; exit 1; }
        qemu-img snapshot -d "$SNAP_NAME" "$DISK"
        echo "Snapshot '$SNAP_NAME' deleted."
        ;;
    *)
        echo "Usage: $0 {list|create|restore|delete} <vm> [snapshot-name]"
        exit 1
        ;;
esac
```

用法：

```bash
chmod +x ~/qemu-vms/*.sh

# 创建 VM
~/qemu-vms/start-vm.sh dev

# 管理快照
~/qemu-vms/snapshot.sh create dev clean-install
~/qemu-vms/snapshot.sh list dev
~/qemu-vms/snapshot.sh restore dev clean-install

# 停止
~/qemu-vms/stop-vm.sh dev
```

---

## 12. 常见问题排查

### VM 启动卡住

```bash
# 1. 确认固件路径正确
ls /opt/homebrew/share/qemu/edk2-aarch64-code.fd

# 2. 加 -nographic 看串口输出，确认卡在哪里

# 3. 检查是否有残留 socket
rm -f /tmp/qemu-monitor.sock

# 4. 清理残留 PID
rm -f /tmp/qemu-*.pid
```

### 网络不通

```bash
# 在 VM 内检查
ip addr                    # 看有没有获取到 IP
ping 10.0.2.2              # ping 默认网关（user 模式）
ping 8.8.8.8               # ping 外网
nslookup google.com        # DNS 测试

# 如果 DNS 不行
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

### 磁盘空间不足

```bash
# 扩容镜像
qemu-img resize myvm.qcow2 60G

# 在 VM 内扩展分区（以 Ubuntu 为例）
sudo parted /dev/vda resizepart 1 100%
sudo resize2fs /dev/vda1     # ext4
# 或
sudo xfs_growfs /            # xfs
```

### 性能差

```bash
# 确认使用了加速器
# 检查命令中有没有 -accel hvf

# 确认用了 virtio 驱动
# 磁盘: -drive if=virtio（不是 if=ide）
# 网络: -device virtio-net-pci（不是 e1000）

# 检查 cache 模式
# 测试环境用 cache=writeback
```

### SSH 连接被拒绝

```bash
# 确认端口映射正确
hostfwd=tcp::2222-:22

# 确认 VM 里 sshd 在运行
sudo systemctl status ssh

# 确认防火墙
sudo ufw allow 22

# 检查 known_hosts 冲突
ssh-keygen -R "[localhost]:2222"
```

### Apple Silicon 上跑 x86 系统

**结论：可以，但非常慢**（纯软件模拟）。

```bash
qemu-system-x86_64 \
  -accel tcg,thread=multi \
  -smp 2 \
  -m 2G \
  ...
```

如果需要跑 x86 容器，用 Rosetta（需要 macOS Ventura+）：

```bash
# 通过 Apple Virtualization Framework 的 Rosetta
# 这不是 QEMU 原生支持的，需要通过 container/OrbStack 等上层工具
```

---

## 速查表

| 需求 | 命令 |
|---|---|
| 创建磁盘 | `qemu-img create -f qcow2 disk.qcow2 40G` |
| 查看磁盘信息 | `qemu-img info disk.qcow2` |
| 调整磁盘大小 | `qemu-img resize disk.qcow2 60G` |
| 格式转换 | `qemu-img convert -f raw -O qcow2 in.raw out.qcow2` |
| 创建快照 | `qemu-img snapshot -c snap1 disk.qcow2` |
| 恢复快照 | `qemu-img snapshot -a snap1 disk.qcow2` |
| 列出快照 | `qemu-img snapshot -l disk.qcow2` |
| SSH 进 VM | `ssh -p 2222 user@localhost` |
| 传文件到 VM | `scp -P 2222 file user@localhost:~` |
| 连接 Monitor | `socat - UNIX-CONNECT:/tmp/qemu-monitor.sock` |
| 优雅关机 | Monitor 中输入 `system_powerdown` |
| 后台运行 | 加 `-daemonize -pidfile /tmp/qemu.pid` |
