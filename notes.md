## 收集cpu 磁盘信息

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

## windows 配置winrm

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

## ansible

ansible 执行 win分组下所有30.21的机器 -> `ansible 'win:&*.30.21' -m win_ping`

比较两列不同

 comm -3 <(sort -u list) <(sort -u list2)

## mysql 授权远程登录及错误问题处理

https://blog.csdn.net/hanhanwanghaha/article/details/105599321


```bash
echo "bind-address = 0.0.0.0" >> /etc/my.cnf
service mysqld restart
```
```sql
CREATE USER 'root'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
flush privileges;
```