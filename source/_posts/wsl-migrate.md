---
title: WSL 切换存储路径
date: 2024-10-17 17:08:08
tags: tools
categories: Posts
---

# WSL 切换存储路径

> WSL默认安装在C盘，使用过程中容易撑满C盘

```bash
# 查看容器信息
wsl -l -v 

# 如果安装了docker则执行
# 先在任务栏把docker-desktop退出
wsl --export docker-desktop-data "D:\wsl\docker-desktop-data.tar"
wsl --unregister docker-desktop-data
wsl --import docker-desktop-data "D:\wsl\docker-desktop-data" "D:\wsl\docker-desktop-data.tar" --version 2

wsl --export docker-desktop "D:\wsl\docker-desktop.tar"
wsl --unregister docker-desktop
wsl --import docker-desktop "D:\wsl\docker-desktop" "D:\wsl\docker-desktop.tar" --version 2

wsl --shutdown
wsl --export Ubuntu "D:\wsl\ubuntu.tar"
wsl --unregister Ubuntu
wsl --import Ubuntu "D:\wsl\ubuntu" "D:\wsl\ubuntu.tar" --version 2
## 配置默认登陆用户
ubuntu config --default-user guest
```