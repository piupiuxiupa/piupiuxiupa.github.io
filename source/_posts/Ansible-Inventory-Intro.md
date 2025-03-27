---
title: Ansible Inventory Intro
date: 2024-07-12 13:35:03
tags: ansible
categories: Posts
---
# ansible inventory 文件基本介绍

> 默认配置路径
>
> `/etc/ansible/hosts`

## 定义主机和组

ansible 支持将同一个主机同时归并到多个不同的组中

```ini
# 可以直接写IP地址
192.168.3.27
# 支持hostname
ntp.example.com
# 支持:+port
192.168.3.27:22
ntp.example.com:222
# []表示一个分组
[ntpserver]
ntp1.example.com
ntp2.example.com
# 表示3-4之间的所有数字,包含3,4
ntp[3:4].example.com
# 表示a-c之间所有数字,包含a,c
ntp[a:c].example.com
```

## 定义主机变量

```ini
[webservers]
web1.example.com http_port=808
maxRequestsPerChild=801 
# 自定义http_port的端口号为808，配置maxRequestsPerChild为801
```

> Ansible其实支持多种方式修改或自定义变量，Inventory是其中的一种修改方式

## 定义组变量

```ini
[groupservers]
web1.example.com
web2.example.com
[groupservers:vars]
ntp_server=ntp.example.com  
# 定义groupservers组中所有主机ntp_server值为ntp.magedu.com
nfs_server=nfs.example.com 
# 定义groupservers组中所有主机nfs_server值为nfs.magedu.com
```

## 定义组嵌套及组变量

```ini
[apache]
httpd1.example.com
httpd2.example.com
[nginx]
ngx1.example.com
ngx2.example.com
[webservers:children]
apache
nginx
[webservers:vars]
ntp_server=ntp.example.com
```

## 多重变量定义

变量除了在inventory中定义,还可以在其他格式的配置文件中定义，如：yml、json

通常在以下四个位置检索

- inventory配置文件
- playbook中vars定义的区域
- roles中vars目录下的文件
- roles同级目录group_vars和hosts_vars目录下的文件

## 其他参数列表

```ini
ansible_ssh_host=
ansible_ssh_user=root
ansible_ssh_pass=xxx
ansible_ssh_port=22
# 指定特有私钥文件
ansible_ssh_private_key_file=
```

> 更多参数查询官网