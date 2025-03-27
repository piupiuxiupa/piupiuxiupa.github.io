---
title: Ansible 批量设置免密登录
date: 2024-08-29 15:07:38
tags: ansible
categories: posts
---

# Ansible 批量设置免密登录

## 1. 准备 inventory && sshkey
  
  首次批量设置免密必然涉及使用带账号密码的inventory文件，
  在设置完后记得修改或删除inventory文件。

  生成公钥和私钥

  ```bash
  # 随机生成公私钥对，ssh-keygen是Linux下认证密钥生成、管理和转换工具，详细用法可参考其man文档
  # 最简单的用法，然后敲回车即可
  ssh-keygen -t rsa
  # -N 必须设置为空，否则ssh时依然需要输入-N设置的密码
  # -C 添加注释
  ssh-keygen -N "" -b 4096 -t rsa -C "root@localhost.com" -f /root/.ssh/root.rsa
  ```

## 2. 编写playbook or 使用ad-hoc

- authorized_key 模块
  ```bash
  ## manage_dir 意思是如果/root没有.ssh文件目录就自动创建.ssh文件目录
  ansible machine -m authorized_key -a  "user=root  key='{{  lookup('file', '/root/.ssh/id_rsa.pub')  }}'  path=/root/.ssh/authorized_keys  manage_dir=yes"
  ```
- copy && shell 模块
  ```bash
  # copy公钥至远程主机/tmp目录下
  ansible machine -m copy -a "src=/root/.ssh/id_rsa.pub dest=/tmp/id_rsa.pub"
  # 添加公钥
  ansible machine -m shell -a "cat /tmp/id_rsa.pub >> /root/.ssh/authorized_keys"
  ```
- 少量主机时可用
  - 需要自己输入密码
  - 主机较多时频繁输入密码较麻烦，主机量少时可以使用
  - 个别新增可用此方法
  - 也可配合expect实现自动输入密码

```bash
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.1.1
```
关于expect用法可查询其他资料，bash脚本可以结合expect使用，从而达到自动批量登录设置免密的效果。

```bash
# ! /usr/bin/expect
# 设置要连接的远程主机IP信息
set IP      [ lindex $argv 0 ]
# 设置要连接的远程主机登录用户
set USER    [ lindex $argv 1 ]
# 设置要连接的远程主机登录用户的密码信息
set PASSWD [ lindex $argv 2 ]
# 设置要执行的命令
set CMD     [ lindex $argv 3 ]
# spawn是expect内部命令，开启ssh连接
spawn ssh $USER@$IP $CMD
# 判断上次执行结果
expect {
    # 如果有yes或no关键字
    "(yes/no)? " {
        # 则输入yes
        send "yes\r"
        # 输入完yes后如果输出结果有：password: 关键字
        expect "password:"
        # 则输入密码文件
        send "$PASSWD\r"
        }
    # 如果上次输出结果有：password: ，则输入密码
    "password:" {send "$PASSWD\r"}
    # 如果上次输出结果有：* to host ，则退出
    "* to host" {exit 1}
    }
expect eof
```

## 3. 验证
```bash
# 将 ip host 写入/etc/hosts文件，如
echo "192.168.1.1 master" >> /etc/hosts
## 实现直接输入master登录远程服务器
ssh master 
```