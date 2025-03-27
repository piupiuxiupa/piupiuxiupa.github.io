---
title: WSL 基础用法
date: 2024-08-04 14:40:41
tags: tools
categories: Posts
---

# WSL 基础用法

## 检查 WSL是否安装

```powershell
## 检查版本
wsl --version
```

![wsl-version](../images/wsl-base/wsl-version.png)

有输出如图则为已安装。

## 安装 WSL

> [旧版安装方式](https://learn.microsoft.com/zh-cn/windows/wsl/install-manual)

安装 WSL 2 之前，必须启用“虚拟机平台”可选功能。

运行如下命令：

```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

必须运行 Windows 10 版本 2004 及更高版本（内部版本 19041 及更高版本）或 Windows 11 才能使用以下命令。 

在管理员模式下打开 PowerShell 或 Windows 命令提示符，输入如下命令，然后重启计算机。

```powershell
# 此命令将启用运行 WSL 并安装 Linux 的 Ubuntu 发行版所需的功能
wsl --install

## 更新内核
## Linux 内核更新包会安装最新版本的 WSL 2 Linux 内核，以便在 Windows 操作系统中运行 WSL
wsl --update

## 设置默认版本
wsl --set-default-version 2
```

## 查看发行版本

```powershell
## 查看可安装发行版
wsl --list --online
wsl -l -o
## 查看已安装发行版
wsl -l -v
```

## 安装一个发行版

```powershell
## 可直接运行下载安装
wsl --install -d <Distribution Name>
wsl --install --web-download -d <Distribution Name>
## 也可直接本地导入
## docker导出的容器tar包可用在wsl
docker export $container -o $container.tar
wsl --import centos D:\wsl\centos\ .\centos.tar
```

> [导入任意发行版](https://learn.microsoft.com/zh-cn/windows/wsl/use-custom-distro)

## 操作虚拟机

使用`wsl -l -v`可看到如图有四个已安装发行版

![installed](../images/wsl-base/installed.png)

```powershell
## 进入centos版本linux
wsl -d centos
```

要设置与 `wsl` 命令一起使用的默认 Linux 发行版，输入 `wsl -s ` 或 `wsl --set-default `。

例如，从 PowerShell/CMD 输入 `wsl -s Debian`，将默认发行版设置为 Debian。 现在从 Powershell 运行 `wsl npm init` 将在 Debian 中运行 `npm init` 命令。

## 修改 WSL配置

1. 机器内配置

   进入虚拟机后，运行`df -h`可以查看到宿主机的各个盘符都已挂载到该机器上。

```bash
## 默认不可用systemctl控制服务，需要WSL版本0.67.6以上
## 默认windows访问wsl linux 以默认用户访问，因此无法访问root用户文件
## 修改为可用systemd及默认root用户登录
cat >> /etc/wsl.conf <<EOF
[boot]
systemd=true
[user]
default=root
EOF
```

2. WSL 全局配置

   开启镜像ip地址，虚拟机地址会变更为宿主机同一个ip。

```bash
cat >>/mnt/c/Users/xxx/.wslconfig <<EOF
[experimental]
networkingMode=mirrored
autoProxy=true
dnsTunneling=true
firewall=true
EOF
```

修改完配置文件后需要运行`wsl --shutdown`，并需要**间隔8秒**后再启动虚拟机。

> [高级设置配置](https://learn.microsoft.com/zh-cn/windows/wsl/wsl-config)

## 删除不需要的发行版

```powershell
wsl --unregister <Distro>
```

