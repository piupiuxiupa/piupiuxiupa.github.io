---
title: Free Notes
date: 2025-03-27 20:43:24
categories: Notes
excerpt: 个人的一些随记，待整理
---

## 收集cpu 磁盘信息

### Linux

```bash
lsblk -db -e 7 -o SIZE -n | awk '{s+=$1} END{printf "%.1fTB", s/1024/1024/1024/1024}'

# CPU信息：Socket(s) 插槽数，Core(s) per socket 每个插槽核心数，Thread(s) per core 每个核心线程数
# CPU名：
lscpu | grep 'Model name'
# 物理核心数：
lscpu | grep -i 'socket' | awk '{print $NF}' | paste -s -d '*' | bc
# 总线程数（逻辑核心数）：
lscpu | grep  '^CPU(s):'|awk '{print $2}'

# 查看网口信息
ethtool eth0 

# 查看GPU
lspci
```

### Windows

```powershell
# 查看网络接口信息
Get-NetAdapter | Select-Object Name, InterfaceDescription, LinkSpeed
# 查看内存信息
Get-CimInstance Win32_ComputerSystem | Select-Object @{Name="TotalMemory(GB)";Expression={[math]::Round($_.TotalPhysicalMemory / 1GB, 2)}}
# 查看每条内存信息
Get-CimInstance Win32_PhysicalMemory | Select-Object Manufacturer, @{Name="Size(GB)";Expression={[math]::Round($_.Capacity / 1GB, 2)}}, Speed
# 查看CPU信息
Get-CimInstance Win32_Processor | Select-Object Name, NumberOfCores, NumberOfLogicalProcessors
# 查看磁盘信息
Get-PhysicalDisk | Select-Object FriendlyName, MediaType, @{Name="Size(GB)";Expression={[math]::Round($_.Size / 1GB, 2)}}
```

## Windows 配置winrm

- python基础环境
- 防火墙放开5985端口

```ps1
# 查看powershell版本
get-host
# 查看执行策略
get-executionpolicy
# 修改执行策略
set-executionpolicy remotesigned
# 启动winrm服务
winrm quickconfig
# 列出该计算机上的所有 WinRM 侦听程序
winrm enumerate winrm/config/listener
# 修改配置
# 允许基本身份验证
winrm set winrm/config/service/auth '@{Basic="true"}' 
# 允许非加密通信
winrm set winrm/config/service '@{AllowUnencrypted="true"}' 
# 允许远程访问
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force
# 打开防火墙端口
New-NetFirewallRule -Name "WinRM" -DisplayName "WinRM" -Enabled True -Protocol TCP -Action Allow -LocalPort 5985
```

## windows 设置密码

如果通过界面设置密码，密码会被显示为*，不好分辨是否包含空格或回车，设置的时候可以点击显示查看下。
如果通过cmd来设置，如:

```powershell
# 此处与linux不同，''也会被视为密码的一部分
net user administrator 'as^df!@ii'
# ^ 是windows命令行的转义符，^d 其实是 d
echo ^d  # 会输出 d
```

## Ansible

```bash
ansible 执行 win分组下所有30.21的机器 -> `ansible 'win:&*.30.21' -m win_ping`

```

## 比较两列不同

comm -3 <(sort -u list) <(sort -u list2)

## mysql 授权远程登录及错误问题处理

[参考链接](https://blog.csdn.net/hanhanwanghaha/article/details/105599321)

```bash
echo "bind-address = 0.0.0.0" >> /etc/my.cnf
service mysqld restart
```

```sql
CREATE USER 'root'@'%' IDENTIFIED BY 'password';
-- 库.表    *.* 所有库所有表
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
flush privileges;
```
