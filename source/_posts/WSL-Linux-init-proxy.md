---
title: WSL启动Linux并初始化及代理配置
date: 2024-08-04 19:45:05
tags: tools
categories: Posts
---

## WSL 启动 Ubuntu

由于22.04 jammy不支持高版本nodejs，所以我们下载24.04 noble版本的Ubuntu。

```powershell
## 查看可下载的发行版
wsl -l -o

wsl --install -d Ubuntu-24.04 
```

![list](../images/wsl-linux-init/available_list.png)

等待下载完成就会自动进入。

### 配置 apt阿里源

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak

## 查看版本codename
cat /etc/os-release | grep UBUNTU_CODENAME
## 如用其他版本，如22.04 jammy，将noble替换为jammy即可
cat >> /etc/apt/sources.list <<EOF
deb http://mirrors.aliyun.com/ubuntu/ noble main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ noble main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ noble-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ noble-updates main restricted universe multiverse
EOF
## 配置完后update
apt update

apt install nodejs npm
```

## WSL 本地导入CentOS

WSL没有CentOS的发行版，因此需要自行导出一个。

centos的tar包从docker容器导出。

```powershell
wsl --import CentOS D:\wsl\centos\ .\centos.tar
wsl -d CentOS

cat >> /etc/wsl.conf <<EOF
[boot]
systemd=true
EOF
```

### 配置 yum阿里源

```bash
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum clean all && yum makecache

yum install initscripts
yum install net-tools
yum install which
yum install sudo
yum install tmux
```

## 配置代理

此处我将自己的WSL网络设置成了`mirrored`，因此在访问一些服务的时候可以通过127.0.0.1来访问。

比如 ssh可以通过`ssh root@127.0.0.1`来远程连接，方便使用其他终端程序；Web服务可以通过http://127.0.0.1:80 访问。

```bash
## 设置网络为镜像网络，完成后关闭虚拟机，等待8秒后再启动
cat >>/mnt/c/Users/xxx/.wslconfig <<EOF
[experimental]
networkingMode=mirrored
EOF

## 配置代理
## 根据自己选择的工具填写端口号
cat >> ~/.bashrc <<EOF
alias proxy='export all_proxy=http://127.0.0.1:7890'
alias unproxy='unset all_proxy'
EOF
source ~/.bashrc
## 运行即可启动代理
proxy

## 配置npm代理
npm config set proxy=http://127.0.0.1:7890
## 默认registry地址，可换成其他国内源
npm config set registry=http://registry.npmjs.org

npm install -g hexo

# 或设置代理
npm config set proxy http://127.0.0.1:7890
npm config set https-proxy http://127.0.0.1:7890
npm config set strict-ssl false
# 取消代理
npm config delete proxy
npm config delete https-proxy
```

## 配置 github加速

在`/etc/hosts`下添加

```
199.96.58.157 github.global.ssl.fastly.net
20.205.243.166 github.com
```

