---
title: Multipass 基本用法及常见问题
date: 2024-07-08 11:02:11
tags: tools
categories: Posts
---
这是一份Multipass基本操作文档，以及包含一些常见问题的解决方法。

Multipass 是一个轻量级的虚拟机管理工具，旨在简化在本地环境中创建和管理 Ubuntu 虚拟机。

参考资料

- [Multipass 官网](https://multipass.run/)
- [Multipass 文档](https://multipass.run/docs)

# 安装Multipass

1. **在 Linux 上安装**:

   ```bash
   sudo snap install multipass
   ```

2. **在 macOS 上安装**:
   使用 Homebrew:

   ```bash
   brew install --cask multipass
   ```

3. **在 Windows 上安装**:
   访问 [Multipass 官网](https://multipass.run/) 下载适用于 Windows 的安装程序，并按照提示完成安装。

## 基本命令

1. **启动一个新的虚拟机**:

   ```bash
   multipass launch --name <instance-name>
   ```

   例如:

   ```bash
   multipass launch --name my-vm
   ```

2. **列出所有虚拟机**:

   ```bash
   multipass list
   ```

3. **进入虚拟机**:

   ```bash
   multipass shell <instance-name>
   ```

   例如:

   ```bash
   multipass shell my-vm
   ```

4. **停止虚拟机**:

   ```bash
   multipass stop <instance-name>
   ```

   例如:

   ```bash
   multipass stop my-vm
   ```

5. **删除虚拟机**:

   ```bash
   multipass delete <instance-name>
   ```

   例如:

   ```bash
   multipass delete my-vm
   ```

6. **清理已删除的虚拟机**:

   ```bash
   multipass purge
   ```

## 进阶用法

1. **指定 Ubuntu 版本**:

   ```bash
   multipass launch --name <instance-name> <ubuntu-version>
   ```

   例如，启动一个 Ubuntu 20.04 的虚拟机:

   ```bash
   multipass launch --name my-vm 20.04
   ```

2. **分配更多资源（CPU、内存、磁盘）**:

   ```bash
   multipass launch --name <instance-name> --cpus <number> --mem <size> --disk <size>
   ```

   例如，分配 2 个 CPU，4GB 内存，20GB 磁盘:

   ```bash
   multipass launch --name my-vm --cpus 2 --mem 4G --disk 20G
   ```

3. **共享文件夹**:

   ```bash
   multipass mount <host-directory> <instance-name>:<target-directory>
   ```

   例如，将本地目录 `/home/user/projects` 挂载到虚拟机的 `/home/ubuntu/projects`:

   ```bash
   multipass mount /home/user/projects my-vm:/home/ubuntu/projects
   ```

4. **运行命令**:
   可以在虚拟机中运行命令而不进入虚拟机：

   ```bash
   multipass exec <instance-name> -- <command>
   ```

   例如，更新虚拟机中的包列表:

   ```bash
   multipass exec my-vm -- sudo apt update
   ```

# 启动前配置

```bash
## 开启mount挂载，方便拿或传文件到服务器，默认false
multipass set local.privileged-mounts=true
## 选择虚拟机驱动，hyperv/virtualbox
## win专业版可用hyperv，家庭版只能virtualbox
multipass set local.driver=hyperv
## 设置默认操作实例
multipass get client.primary-name=primary
```

# 启动实例

```bash
multipass launch -n $name -d 10G -m 2G -c 2 --bridged --mount <local-path>:<instance-path>
```

> 如果启动不了可以尝试重启multipass

# 调整实例配置

```bash
multipass stop handsome-ling
multipass set local.handsome-ling.cpus=4
multipass set local.handsome-ling.disk=60G
multipass set local.handsome-ling.memory=7G
```

# 删除

```bash
## 删除并清除
## delete 相当于把实例转为删除状态 
## -p or --purge 删除并清除
multipass delete $instance -p
## purge
multipass purge $instance
```

# IPv4 N/A

- 重启实例

  ```bash
  multipass restart $instance
  ```

- 停止后再启动实例

  ```bash
  multipass stop $instance
  multipass start $instance
  ```

- 重启multipass服务

  - 可直接杀进程

  ```powershell
  taskkill /f /pid $pid
  ```

- 状态suspending ，尝试suspend实例

  ```bash
  multipass suspend $instance
  ```

- 状态suspended， shell连接即可

  ```bash
  multipass shell $instance
  ```

  
