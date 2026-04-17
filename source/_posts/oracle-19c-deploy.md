---
title: CentOS 8 部署 Oracle 19c
date: 2026-04-17 16:51:20
tags: database
categories: Notes
---

## 环境说明

| 项目 | 说明 |
|------|------|
| 操作系统 | CentOS 8.x (x86_64) |
| 安装包 | [LINUX.X64_193000_db_home.zip](https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html) |
| Oracle 版本 | 19.3.0.0.0 |
| Oracle Home | /u01/app/oracle/product/19c/dbhome_1 |
| Oracle Base | /u01/app/oracle |

## 1 系统准备

### 1.1 关闭防火墙和 SELinux

```bash
systemctl stop firewalld
systemctl disable firewalld

# 临时关闭 SELinux
setenforce 0

# 永久关闭 SELinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

### 1.2 配置 /etc/hosts

确保主机名能正确解析：

```bash
# 查看 IP 和主机名
ip addr
hostname

# 编辑 /etc/hosts，添加如下行（替换为实际值）
echo "192.168.1.100 oracledb" >> /etc/hosts
```

### 1.3 配置内核参数

创建 `/etc/sysctl.d/99-oracle.conf`：

```ini
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
kernel.panic_on_oops = 1
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
vm.swappiness = 0
net.core.somaxconn=1024
net.ipv4.tcp_max_tw_buckets=5000
net.ipv4.tcp_max_syn_backlog=1024
net.ipv4.conf.all.rp_filter=2
net.ipv4.conf.default.rp_filter=2
fs.aio-max-nr = 1048576
net.ipv4.ip_local_port_range=9000 65500
```

> **kernel.shmmax** 建议设置为物理内存的一半（字节）。例如 16GB 内存设置为 `8589934592`。

生效：

```bash
sysctl --system
```

### 1.4 配置资源限制

创建 `/etc/security/limits.d/99-oracle.conf`：

```ini
oracle soft nproc 16384
oracle hard nproc 16384
oracle soft nofile 1024
oracle hard nofile 65536
oracle soft stack 10240
oracle hard stack 32768
oracle hard memlock 134217728
oracle soft memlock 134217728
```

> **memlock** 的单位为 KB，上面的值约 128GB，根据实际物理内存调整。

### 1.5 创建用户和组

```bash
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 oper
groupadd -g 54324 backupdba
groupadd -g 54325 dgdba
groupadd -g 54326 kmdba
groupadd -g 54330 racdba

useradd -u 54321 -g oinstall -G dba,oper,backupdba,dgdba,kmdba,racdba oracle

# 设置 oracle 用户密码
passwd oracle
```

### 1.6 创建目录结构

```bash
mkdir -p /u01/app/oracle/product/19c/dbhome_1
mkdir -p /u01/app/oracle/oradata
mkdir -p /u01/app/oracle/fast_recovery_area
mkdir -p /u01/app/oracle/inventory

chown -R oracle:oinstall /u01/app/oracle
chmod -R 775 /u01/app/oracle
```

### 1.7 配置 Oracle 用户环境

切换到 oracle 用户：

```bash
su - oracle
```

编辑 `~/.bash_profile`，追加以下内容：

```bash
# Oracle Environment
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19c/dbhome_1
export ORACLE_SID=orcl
export PATH=$ORACLE_HOME/bin:$PATH

# 防止安装界面中文乱码（如系统 locale 为中文）
export LC_ALL=C
```

生效：

```bash
source ~/.bash_profile
```

## 2 安装依赖包

CentOS 8 的包名与 Oracle 官方文档不完全一致，需要手动解决依赖问题：

```bash
dnf install -y bc \
  binutils \
  compat-openssl10 \
  elfutils-libelf \
  fontconfig \
  glibc \
  glibc-devel \
  ksh \
  libaio \
  libaio-devel \
  libXrender \
  libX11 \
  libXau \
  libXi \
  libXtst \
  libgcc \
  libnsl \
  libstdc++ \
  libstdc++-devel \
  libcbor \
  libfido2 \
  make \
  policycoreutils \
  policycoreutils-python-utils \
  smartmontools \
  sysstat \
  unzip \
  wget
```

> 如果某些包在 CentOS 8 中找不到，可尝试 `dnf provides <文件名>` 反查，或从 EPEL 仓库获取。

### 2.1 额外处理：libnsl2 和 compat-libpthread

CentOS 8 默认仓库可能缺少部分兼容库，按需安装：

```bash
# 启用 PowerTools 仓库
dnf config-manager --set-enabled powertools
# 或
dnf config-manager --set-enabled PowerTools

dnf install -y libnsl2 libnsl2-devel
```

## 3 安装 Oracle 19c

### 3.1 上传并解压安装包

```bash
# 将 LINUX.X64_193000_db_home.zip 上传至服务器后，解压到 ORACLE_HOME
unzip /path/to/LINUX.X64_193000_db_home.zip -d $ORACLE_HOME
```

> **重要**：解压目标目录就是 `$ORACLE_HOME`，不是上级目录。

### 3.2 静默安装（无图形界面）

#### 3.2.1 创建响应文件

创建 `/tmp/db_install.rsp`：

```ini
####################################################################
## Oracle Database 19c 安装响应文件
####################################################################

oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v19.0.0

# 安装选项: INSTALL_DB_SWONLY（仅安装软件）
oracle.install.option=INSTALL_DB_SWONLY

# UNIX 组名
UNIX_GROUP_NAME=oinstall

# Oracle Inventory 目录
INVENTORY_LOCATION=/u01/app/oracle/inventory

# Oracle Base
ORACLE_BASE=/u01/app/oracle

# Oracle Home
ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1

# 安装版本: EE(企业版) / SE2(标准版)
oracle.install.db.InstallEdition=EE

# 不安装 Oracle Root 配置
oracle.install.db.OSDBA_GROUP=dba
oracle.install.db.OSOPER_GROUP=oper
oracle.install.db.OSBACKUPDBA_GROUP=backupdba
oracle.install.db.OSDGDBA_GROUP=dgdba
oracle.install.db.OSKMDBA_GROUP=kmdba
oracle.install.db.OSRACDBA_GROUP=racdba

# 安全更新（不需要）
oracle.installer.autoupdates.option=SKIP_UPDATES
```

#### 3.2.2 执行静默安装

```bash
cd $ORACLE_HOME

# 由于19c不识别centos 8系统的问题需要设置变量
export CV_ASSUME_DISTID=OEL7.8
# 以 oracle 用户执行
# 或 -ignorePrereq
./runInstaller -silent -ignorePrereqFailure -responseFile /tmp/db_install.rsp
# 或
./runInstaller -silent -responseFile /u01/app/oracle/product/19c/dbhome_1/install/response/db_install.rsp \
oracle.install.option=INSTALL_DB_SWONLY \
UNIX_GROUP_NAME=oinstall \
INVENTORY_LOCATION=/u01/app/oraInventory \
ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1 \
ORACLE_BASE=/u01/app/oracle \
oracle.install.db.InstallEdition=EE \
oracle.install.db.OSDBA_GROUP=dba \
oracle.install.db.OSBACKUPDBA_GROUP=backupdba \
oracle.install.db.OSDGDBA_GROUP=dgdba \
oracle.install.db.OSKMDBA_GROUP=kmdba \
oracle.install.db.OSRACDBA_GROUP=racdba \
oracle.install.db.OSOPER_GROUP=oper
```

> `-ignorePrereqFailure` 跳过先决条件检查失败项（如内存不足、swap 不够等，仅限测试环境）。
> 生产环境建议逐项解决警告后再安装。

#### 3.2.3 以 root 执行脚本

安装程序会提示以 root 用户执行两个脚本：

```bash
# 切换到 root 用户
su -

# 执行以下脚本（路径根据实际输出）
/u01/app/oracle/inventory/orainstRoot.sh
/u01/app/oracle/product/19c/dbhome_1/root.sh
```

执行成功后，回到 oracle 用户继续。

### 3.3 图形界面安装（可选）

如果服务器有图形环境或通过 X11 转发：

```bash
# 设置 DISPLAY（X11 转发场景，IP 为本机 IP）
export DISPLAY=192.168.1.100:0.0

# 也可使用 X forwarding
# ssh -X oracle@server

cd $ORACLE_HOME
./runInstaller
```

按向导操作：
1. 选择 **Set Up Software Only**（仅安装软件）
2. 选择 **Single instance database installation**
3. 选择版本（Enterprise Edition）
4. 确认 Oracle Base 和 Oracle Home 路径
5. 确认操作系统组
6. 以 root 执行脚本

## 4 创建数据库

### 4.1 使用 DBCA 静默建库

创建 `/tmp/dbca.rsp`：

```ini
responseFileVersion=/oracle/assistants/rspfmt_dbca_response_schema_v19.0.0

# 操作类型: createDatabase
operationType=createDatabase

# 模板名（通用模板）
templateName=General_Purpose.dbc

# 数据库全局标识
gdbName=orcl

# SID
sid=orcl

# 数据库字符集
characterSet=AL32UTF8

# 国家字符集
nationalCharacterSet=AL16UTF16

# Sys 和 System 密码（生产环境请使用强密码）
sysPassword=Oracle_19c_Sys
systemPassword=Oracle_19c_System

# 是否创建容器数据库
createAsContainerDatabase=true

# 可插拔数据库名
numberOfPDBs=1
pdbName=ORCLPDB1

# PDB 管理员密码
pdbAdminPassword=Oracle_19c_PdbAdmin

# 数据库文件存储类型: FS（文件系统）
databaseType=MULTIPURPOSE

# 数据文件目录
datafileDestination=/u01/app/oracle/oradata

# 闪回恢复区
recoveryAreaDestination=/u01/app/oracle/fast_recovery_area

# 内存分配（自动，按物理内存比例）
memoryPercentage=40

# 是否启用归档
storageType=FS
```

执行建库：

```bash
dbca -silent -createDatabase -responseFile /tmp/dbca.rsp
# 或
# 单实例数据库
dbca -silent -createDatabase \
-gdbname orcl \
-sid orcl \
-templateName General_Purpose.dbc \
-characterSet AL32UTF8 \
-sysPassword Oracle123 \
-systemPassword Oracle123 \
-datafileDestination /u01/app/oracle/oradata
```

> 如果不想创建 CDB，设置 `createAsContainerDatabase=false` 并移除 PDB 相关参数。

### 4.2 常见 DBCA 问题

**问题：ORA-00845 MEMORY_TARGET 报错**

```bash
# 查看 /dev/shm 大小
df -h /dev/shm

# /dev/shm 应大于 SGA 目标大小，如需调整：
mount -o remount,size=8G /dev/shm

# 持久化，编辑 /etc/fstab
echo "tmpfs /dev/shm tmpfs defaults,size=8G 0 0" >> /etc/fstab
```

**问题：dbca 日志排查位置**

```
$ORACLE_BASE/cfgtoollogs/dbca/<SID>/
```

## 5 配置监听

### 5.1 使用 netca 静默配置

```bash
netca -silent -responseFile $ORACLE_HOME/assistants/netca/netca.rsp
```

或手动创建 `$ORACLE_HOME/network/admin/listener.ora`：

```ini
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )
```

创建 `$ORACLE_HOME/network/admin/tnsnames.ora`：

```ini
ORCL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = oracledb)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCL)
    )
  )

ORCLPDB1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = oracledb)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCLPDB1)
    )
  )
```

### 5.2 启动/停止监听

```bash
# 启动
lsnrctl start

# 查看状态
lsnrctl status

# 停止
lsnrctl stop
```

## 6 配置开机自启动

### 6.1 修改 oratab

编辑 `/etc/oratab`，将最后一行的 `N` 改为 `Y`：

```
ORCL:/u01/app/oracle/product/19c/dbhome_1:Y
```

### 6.2 创建 systemd 服务

创建 `/etc/systemd/system/oracle.service`：

```ini
[Unit]
Description=Oracle Database 19c
After=network.target

[Service]
Type=forking
User=oracle
Group=oinstall
Restart=no

ExecStart=/u01/app/oracle/product/19c/dbhome_1/bin/dbstart /u01/app/oracle/product/19c/dbhome_1
ExecStop=/u01/app/oracle/product/19c/dbhome_1/bin/dbshut /u01/app/oracle/product/19c/dbhome_1

[Install]
WantedBy=multi-user.target
```

启用服务：

```bash
systemctl daemon-reload
systemctl enable oracle
systemctl start oracle
```

## 7 常用操作

### 7.1 连接数据库

```bash
# 本地连接 CDB root
sqlplus / as sysdba

# 连接 PDB
sqlplus sys/Oracle_19c_Sys@ORCLPDB1 as sysdba

sqlplus system/password
```

### 7.2 数据库启停

```sql
-- 启动
SQL> startup;

-- 关闭
SQL> shutdown immediate;
```

### 7.3 PDB 管理

```sql
-- 查看所有 PDB
SQL> show pdbs;

-- 切换到 PDB
SQL> alter session set container = ORCLPDB1;

-- 打开 PDB
SQL> alter pluggable database ORCLPDB1 open;

-- 保存 PDB 状态（开机自动打开）
SQL> alter pluggable database ORCLPDB1 save state;

-- 关闭 PDB
SQL> alter pluggable database ORCLPDB1 close;
```

### 7.4 查看数据库版本和状态

```sql
-- 版本
SQL> select * from v$version;

-- 实例状态
SQL> select status from v$instance;

-- 数据库字符集
SQL> select userenv('language') from dual;
```

## 8 验证安装

安装完成后，按以下步骤验证：

```bash
# 1. 检查监听状态
lsnrctl status

# 2. 登录数据库
sqlplus / as sysdba

# 3. 确认实例状态
SQL> select status from v$instance;
# 期望输出: STATUS = OPEN

# 4. 确认 PDB 状态
SQL> show pdbs;
# 期望输出: ORCLPDB1 状态为 READ WRITE

# 5. 确认版本
SQL> select banner_full from v$version;
# 期望输出: Oracle Database 19c ...
```

## 9 故障排查

### 9.1 安装阶段常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `PRVG-13601` | CLONE 阶段报错 | 检查 ORACLE_HOME 权限，确保 `chown -R oracle:oinstall $ORACLE_HOME` |
| `ORA-12547` | TNS 丢失联系 | 检查监听是否启动，`lsnrctl start` |
| `libclntsh.so` not found | 解压路径错误 | 确认 zip 解压到 `$ORACLE_HOME` 内 |
| `DISPLAY` not set | 图形模式找不到显示 | 使用静默安装或配置 X11 转发 |
| `PRVF-7532` | 缺少依赖包 | 对照第 2 节安装所有依赖 |

### 9.2 日志位置

```
安装日志:   /u01/app/oracle/inventory/logs/
DBCA 日志:  /u01/app/oracle/cfgtoollogs/dbca/<SID>/
Alert 日志: /u01/app/oracle/diag/rdbms/<db_name>/<SID>/trace/alert_<SID>.log
监听日志:   /u01/app/oracle/diag/tnslsnr/<hostname>/listener/alert/log.xml
```

## 10 CDB 与 PDB 概念说明

### 10.1 什么是 CDB 和 PDB

类比 Docker 来理解：

| 概念 | 类比 | 说明 |
|------|------|------|
| **CDB**（Container Database） | Docker 引擎 | 容器数据库，提供共享的内存、进程、管理基础设施 |
| **PDB**（Pluggable Database） | Docker 容器 | 可插拔数据库，逻辑上完全独立的数据库 |

CDB 内部结构：

```
CDB (orcl)
├── CDB$ROOT   ← 根容器，存系统级元数据、公共用户、公共角色
├── PDB$SEED   ← 种子容器，只读模板，创建新 PDB 时从这里复制
└── ORCLPDB1   ← 你实际的业务数据库
```

### 10.2 为什么需要 CDB

PDB 不是独立实例，它**依赖 CDB 提供的共享基础设施**：

- **SGA / 后台进程** — 只有一套，所有 PDB 共享内存和进程，省资源
- **系统元数据** — Oracle 内核的字典表只在 CDB root 存一份，PDB 里只存自己的业务数据
- **公共管理** — 打补丁、升级、备份可以在 CDB 层面一次搞定，不用逐个 PDB 操作
- **资源控制** — CDB 统一管理内存/CPU，可以给每个 PDB 分配配额

简单说：**CDB 是引擎，PDB 是数据库。**

### 10.3 Non-CDB vs CDB

| | Non-CDB（传统模式） | CDB + PDB（多租户模式） |
|---|---|---|
| 架构 | 传统单数据库 | 一个 CDB root 下挂多个 PDB |
| 多租户 | 不支持 | 支持，可插拔/拔出 |
| 资源隔离 | 无 | PDB 间可隔离 CPU/内存 |
| 19c 默认 | 否 | 是（Oracle 19c 默认创建 CDB） |
| 许可 | 无限制 | **不超过 3 个 PDB 时免费**（不买 Multitenant 许可） |
| 适用场景 | 只需要一个数据库 | 需要多个独立数据库 |

创建 CDB + PDB：

```bash
dbca -silent -createDatabase \
  -gdbname orcl \
  -sid orcl \
  -templateName General_Purpose.dbc \
  -characterSet AL32UTF8 \
  -sysPassword Oracle123 \
  -systemPassword Oracle123 \
  -datafileDestination /u01/app/oracle/oradata \
  -createAsContainerDatabase true \
  -numberOfPDBs 1 \
  -pdbName orclpdb1 \
  -pdbAdminPassword Oracle123
```

创建 Non-CDB（不加 CDB 相关参数即可）：

```bash
dbca -silent -createDatabase \
  -gdbname orcl \
  -sid orcl \
  -templateName General_Purpose.dbc \
  -characterSet AL32UTF8 \
  -sysPassword Oracle123 \
  -systemPassword Oracle123 \
  -datafileDestination /u01/app/oracle/oradata
```

### 10.4 DBCA 创建命令参数说明

| 参数 | 说明 |
|------|------|
| `-createAsContainerDatabase true` | 创建为容器数据库（CDB） |
| `-numberOfPDBs 1` | 创建几个 PDB，设为 3 则自动创建 orclpdb1、orclpdb2、orclpdb3 |
| `-pdbName orclpdb` | PDB 名称前缀，多个时自动加编号 |
| `-pdbAdminPassword Oracle123` | PDB 管理员（PDBADMIN 用户）的密码 |

> 建议不要把 PDB 命名为与 CDB 相同的名字（如都叫 orcl），容易混淆。

### 10.5 Non-CDB 转 CDB

如果建库时没有创建 CDB，需要转换为 CDB + PDB 架构：

**方案一：无业务数据 → 删库重建（推荐）**

```bash
dbca -silent -deleteDatabase -sourceDB orcl

dbca -silent -createDatabase \
  -gdbname orcl \
  -sid orcl \
  -templateName General_Purpose.dbc \
  -characterSet AL32UTF8 \
  -sysPassword Oracle123 \
  -systemPassword Oracle123 \
  -datafileDestination /u01/app/oracle/oradata \
  -createAsContainerDatabase true \
  -numberOfPDBs 1 \
  -pdbName orclpdb1 \
  -pdbAdminPassword Oracle123
```

**方案二：有数据要保留 → 将 Non-CDB 插入到新 CDB**

```sql
-- 1. 在原 Non-CDB 中，以 read only 打开
shutdown immediate;
startup mount;
alter database open read only;

-- 2. 生成 PDB 描述文件
begin
  dbms_pdb.describe(pdb_descr_file => '/tmp/orcl_pdb.xml');
end;
/

-- 3. 关闭原库
shutdown immediate;
```

```bash
# 4. 创建一个新的 CDB（不含 PDB）
dbca -silent -createDatabase \
  -gdbname newcdb \
  -sid newcdb \
  -templateName General_Purpose.dbc \
  -characterSet AL32UTF8 \
  -sysPassword Oracle123 \
  -systemPassword Oracle123 \
  -datafileDestination /u01/app/oracle/oradata \
  -createAsContainerDatabase true \
  -numberOfPDBs 0
```

```sql
-- 5. 在新 CDB 中，把原库作为 PDB 插入
create pluggable database orclpdb1 using '/tmp/orcl_pdb.xml' copy;
alter pluggable database orclpdb1 open;
alter pluggable database orclpdb1 save state;

-- 6. 运行数据字典升级脚本（耗时较长）
alter session set container=orclpdb1;
@?/rdbms/admin/noncdb_to_pdb.sql
```

### 10.6 多 CDB 与多 PDB

**一台服务器可以运行多个 CDB**，每个 CDB 各自拥有独立的实例、内存（SGA）、后台进程：

```bash
# 第一个 CDB
dbca -silent -createDatabase -gdbname cdb1 -sid cdb1 ...

# 第二个 CDB
dbca -silent -createDatabase -gdbname cdb2 -sid cdb2 ...
```

但一般**不推荐多个 CDB**，每个 CDB 都要吃一份内存。推荐架构：

```
一台服务器
└── 一个 CDB（一套内存、一套进程）
    ├── PDB: ERP
    ├── PDB: CRM
    ├── PDB: 测试环境
    └── PDB: 开发环境
```

在已有 CDB 中追加 PDB：

```sql
-- 从种子库创建新 PDB
create pluggable database hrpdb admin user hradmin identified by Oracle123;
alter pluggable database hrpdb open;
alter pluggable database hrpdb save state;

-- 从已有 PDB 克隆
alter pluggable database bizpdb1 open;
create pluggable database bizpdb2 from bizpdb1;
alter pluggable database bizpdb2 open;
```

### 10.7 PDB 与 Schema 的关系

**PDB 就是数据库，下面没有子库了。** 每个 PDB 内部就是正常的数据库结构：表空间、用户、表、视图、索引。要在 PDB 里"分库"，靠的是用户/Schema 隔离：

```
连接到 orclpdb1（一个 PDB = 一个数据库）
└── Schema（模式）
    ├── SYS          ← 系统自带
    ├── SYSTEM       ← 系统自带
    ├── HR           ← 业务用户，下面有表、视图、索引...
    ├── ORDER_APP    ← 业务用户
    └── REPORT_APP   ← 业务用户
```

| 概念 | 对应关系 |
|------|---------|
| 数据库（类似 MySQL 的 database） | PDB |
| 模式（Schema） | 一个 Oracle 用户拥有的所有对象 |

Oracle 中 **Schema 和用户是一一对应的**，创建一个用户就自动有一个同名的 Schema。

### 10.8 使用 DBeaver 连接

DBeaver 连接时指定的服务名决定了你进入的是哪个 PDB。不同 PDB 需要建不同的连接：

| 连接服务名 | 进入的 PDB | 看到的 Schema |
|------------|-----------|---------------|
| `orclpdb1` | orclpdb1  | 该 PDB 下的所有用户 |
| `orclpdb2` | orclpdb2  | 该 PDB 下的所有用户 |

DBeaver 中的"模式"就是 Schema，不是 PDB。切换"数据库"的做法是建多个连接，分别连不同的 PDB 服务名。
