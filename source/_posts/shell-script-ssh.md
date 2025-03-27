---
title: Shell script ssh & linux cmd tips
date: 2025-03-05 13:22:54
tags: linux
categories: Posts
---
# 脚本批量ssh登录

- 可以通过 expect 实现
- 可以通过 sshpass 实现

## expect 脚本

后续补充

## sshpass 脚本

```bash
#!/bin/bash

while IFS=':' read -r ip uname passwd
do
    echo "ssh $ip $uname $password"
    sshpass -p "$passwd" ssh -o StrictHostKeyChecking=no -n "$uname@$ip" < /dev/null
    sleep 1
done < info.txt
```

> **注意事项**
>
> ssh 会在登录后读取所有标准输入，因此会导致循环 read 后面的内容被 ssh 吞掉，导致只会登录一次
>
> 因此需要用 ssh -n选项或者在最后使用 < /dev/null

## linux command tips

```bash
# rpm -q 只能按开头查询， 如 rpm -q zabbix-agent 只能查到zabbix-agent包，查不到 pcp-export-zabbix-agent这样的包
ps -Fu zabbix # 按用户查找进程
cp /tmp/test.sh{,.bak} # 复制文件并重命名，这样不用再多敲一段路径

## 这段sed基本囊括常用写法，-i.bak 修改的同时生成一个.bak后缀的未修改版文件，-e 后接 sed 表达式，\1表示第一个括号内匹配的内容
sed -i.bak -e "s/\(Server=\|ServerActive=\)[^ ]*/\1192.168.4.4/g" \
    -e "s/\(^Hostname=\)[^.]*/\1"$(ip a | grep inet | grep brd | awk '{print $2}'| head -1 | cut -d/ -f1)"/g" \
    -e "/# HostMetadata=/a HostMetadata=Linux" \
    -e "/# HostMetadataItem=/a HostMetadataItem=Linux" \
    /etc/zabbix/zabbix_agentd.conf

# test命令做逆判断会出现退出状态码为非0的情况，比如
ls
test $? -ne 0 && { echo "failed"; exit 1; } 
# 由于ls命令执行成功，test $? -ne 0 为假，test命令本身返回非零值
# 如果这个语句作为整个脚本的最后一句，会导致即使脚本实际执行成功，但最后脚本的退出码也为非0
# 改成顺逻辑即可
test $? -eq 0 || { echo "failed"; exit 1; }
```
