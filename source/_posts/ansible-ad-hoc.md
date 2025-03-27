---
title: Ansible ad-hoc简介
date: 2024-07-12 15:59:04
tags: ansible
categories: Posts
---
# Ansible Ad-Hoc
Ad-Hoc形式应用场景主要在临时执行一些操作时会用到，更注重解决一些简单或临时任务，复杂操作会使用playbook来实现。
比如想临时传一个文件到被控机上就可以使用ad-hoc形式来完成。
```bash
ansible all -m copy -a "src=/root/monitor.sh dest=/root/ backup=yes"
```
## 常用选项
```bash
# -v 可使用-vvv输出更详细内容
# -i 指定inventory
# -f 指定并发线程数
# --private-key 指定密钥文件
# -m 指定执行模块
# -M 指定模块存放路径
# -a 模块参数，默认command模块
# -t 输出信息到指定目录，文件以主机名命名
# -T 指定最大超时时间
# -B 后台执行命令，超过指定时间后中止执行的任务
# -P 定期以指定时间间隔返回后台任务进度
# --list-hosts 列出符合条件的主机，不执行任何命令
ansible -i hosts all -f 20 -B 10 -P 2 -T 1 -m shell -a "df -h" -vvv
ansible all --list-hosts
```
success 表示命令执行成功，changed 是否对主机做出变更
> 建议并发数配置为CPU核数偶数倍就行。如4C 8G的机器，最多并发20个线程
## ansible-doc
如果不知道有哪些模块以及模块有哪些参数可用，就可使用此命令查看
```bash
ansible-doc -l
ansible-doc -s shell
```
- `-l` 选项列出所有可用模块
- `-s` 只显示playbook说明的代码段
## 指定主机执行任务
```bash
# --limit 
ansible all -m shell -a "ls -l" --limit "192.168.3.27"
# 指定IP
ansible 192.168.3.27 -m shell -a "ls -l"
# : 分隔符指定多台机器做变更，但""必须使用
ansible "192.168.3.27:192.168.33.22" -m shell -a "ls -l"
# * 通配符匹配多台机器
ansible 192.168.3.* -m shell -a "ls -l"
```

