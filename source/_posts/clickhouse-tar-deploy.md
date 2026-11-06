---
title: ClickHouse 部署实战：tar 包单机生产部署与配置管理
date: 2026-11-06 14:15:06
tags: [clickhouse, olap, 运维]
categories: Posts
excerpt: 以 tar 包部署为主线的 ClickHouse 单机生产落地手册：官方 tgz 安装流程、config.d/users.d 配置覆盖、数据目录迁移、用户权限管理、建表与 SQL 执行，附集群概览和一份踩坑排查表。
---

日志分析、宽表聚合这类场景，行存数据库撑不住之后就得 OLAP 引擎上场，ClickHouse 是自建路线里最省心的一支：单机就能吃下可观的日志写入，SQL 生态也齐全。这篇是部署落地手册，以 **tar 包部署为主线**——不依赖发行版的包管理器，版本完全自控，和系统里其他软件互不打扰。apt / rpm / Docker 的路径也保留，场景不同各取所需。

一条贯穿全文的原则：**配置不要直接改 `config.xml` / `users.xml`，在 `config.d/`、`users.d/` 下放独立覆盖文件**，升级时才不会被默认配置冲掉。

## 部署前检查清单

| 项目 | 要求 | 说明 |
|---|---|---|
| Swap | 禁用，或 `vm.swappiness=1` | ClickHouse 高度依赖内存缓存，Swap 会严重拖慢查询 |
| 磁盘 | 二进制预留约 2.5 GB | 数据盘强烈推荐 SSD，机械盘随机读会是瓶颈 |
| CPU / 内存 | 最低 4 核 / 8 GB | 生产建议 16 GB 内存起步，内存直接决定查询速度 |
| 文件句柄 | `nofile` ≥ 262144 | 避免高并发下 "Too many open files" |
| 端口 | 8123 / 9000 按需放行 | 9009 是集群节点通信端口，单机不用放 |

## tar 包部署（本文主线）

官方对 tgz 的定位是：所有装不了 deb/rpm 的 Linux 发行版，用官方预编译的 tgz 归档。压缩包从 [packages.clickhouse.com/tgz/stable](https://packages.clickhouse.com/tgz/stable/) 下载，解压后跑一遍 `doinst.sh` 安装脚本即可。

**第一步：确定版本号。** 官方脚本抓 GitHub 上的最新版本：

```bash
LATEST_VERSION=$(curl -s https://raw.githubusercontent.com/ClickHouse/ClickHouse/master/utils/list-versions/version_date.tsv \
  | grep -Eo '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | sort -V -r | head -n 1)
export LATEST_VERSION
```

生产环境我更建议固定版本：直接去 tgz 仓库里挑一个 LTS 版本号（如 `25.3.x.x`）手动赋值，升级节奏自己说了算。

**第二步：下载组件压缩包。** x86_64 机器 `ARCH=amd64`，ARM 用 `aarch64`：

```bash
ARCH=amd64
for PKG in clickhouse-common-static clickhouse-server clickhouse-client
do
  curl -fO "https://packages.clickhouse.com/tgz/stable/$PKG-$LATEST_VERSION-${ARCH}.tgz" \
  || curl -fO "https://packages.clickhouse.com/tgz/stable/$PKG-$LATEST_VERSION.tgz"
done
```

官方循环里还有 `clickhouse-common-static-dbg`（调试符号，700 MB 上下）和 `clickhouse-keeper`（单机用不到），没有 gdb 分析 core dump 的需求可以都不下。

**第三步：解压并执行安装脚本。**

```bash
tar -xzvf "clickhouse-common-static-$LATEST_VERSION-${ARCH}.tgz"
sudo "clickhouse-common-static-$LATEST_VERSION/install/doinst.sh"

tar -xzvf "clickhouse-server-$LATEST_VERSION-${ARCH}.tgz"
sudo "clickhouse-server-$LATEST_VERSION/install/doinst.sh" configure

tar -xzvf "clickhouse-client-$LATEST_VERSION-${ARCH}.tgz"
sudo "clickhouse-client-$LATEST_VERSION/install/doinst.sh"
```

`doinst.sh` 不是"绿色解压"，它会做完整安装：二进制放到 `/usr/bin/`，默认配置落到 `/etc/clickhouse-server/`（含 `config.d/`、`users.d/`），数据目录 `/var/lib/clickhouse/`，日志 `/var/log/clickhouse-server/`，并创建 `clickhouse` 用户和组。装完的目录布局和 deb/rpm 完全一致，所以后文的配置覆盖、数据迁移、systemd 管理对 tar 部署同样适用。

**第四步：启动、自启、验证。**

```bash
sudo /etc/init.d/clickhouse-server start   # systemd 机器也可 systemctl start clickhouse-server
sudo systemctl enable clickhouse-server

clickhouse-client --query "SELECT version()"
```

**升级**：下载新版本压缩包、重新解压执行一遍 `doinst.sh` 即可替换二进制。数据目录独立于安装包，不在覆盖范围内；动手前照常备份数据和 `config.d/` 覆盖文件。

## 包管理器部署（备选）

apt 源（Debian/Ubuntu）：

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/clickhouse-keyring.gpg

ARCH=$(dpkg --print-architecture)
echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg arch=${ARCH}] \
https://packages.clickhouse.com/deb stable main" \
  | sudo tee /etc/apt/sources.list.d/clickhouse.list
sudo apt-get update
sudo apt-get install -y clickhouse-server clickhouse-client
```

rpm 源（RHEL/CentOS/Rocky/AlmaLinux）：

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo
sudo yum install -y clickhouse-server clickhouse-client
```

装完启动都一样：

```bash
sudo systemctl enable --now clickhouse-server
clickhouse-client   # 设了密码则加 --password
```

macOS 上 Homebrew 有社区维护的 formula，仅建议本地开发验证，不进生产。

## Docker 部署（开发测试与快速验证）

```bash
sudo mkdir -p /data/clickhouse
docker run -d \
  --name clickhouse-server \
  -p 8123:8123 -p 9000:9000 \
  --ulimit nofile=262144:262144 \
  -v /data/clickhouse:/var/lib/clickhouse \
  clickhouse/clickhouse-server:latest
```

两个注意点：`--ulimit` 必须显式给，容器的 nofile 上限继承自 dockerd，改宿主机 limits.conf 对它无效；挂载的宿主机目录要先 `mkdir -p` 真实存在，否则 Docker 会自作主张建占位文件（见文末排查表）。进容器用客户端：`docker exec -it clickhouse-server clickhouse-client`。

## 关键配置

配置文件在 `/etc/clickhouse-server/`，核心是 `config.xml` 和 `users.xml`，但改动都走 `config.d/`、`users.d/` 覆盖。

### 开放远程监听

默认只听 `localhost`，远程连不上十有八九是它：

```xml
<!-- /etc/clickhouse-server/config.d/listen.xml -->
<clickhouse>
    <listen_host>0.0.0.0</listen_host>
</clickhouse>
```

改完 `systemctl restart clickhouse-server`。没配认证就把 9000 暴露公网等于开门揖盗，防火墙限制可信网段 + 密码，缺一不可。

### 端口

| 配置参数 | 默认端口 | 用途 |
|---|---|---|
| `<http_port>` | 8123 | HTTP 接口，JDBC/ODBC、Web 客户端、Prometheus 指标 |
| `<tcp_port>` | 9000 | 原生 TCP，`clickhouse-client` 和大多数语言驱动 |
| `<interserver_http_port>` | 9009 | 节点间数据交换与复制，集群必须互通且各节点一致 |
| `<https_port>` | 8443 | HTTP 的 TLS 版本 |
| `<tcp_port_secure>` | 9440 | 原生 TCP 的 TLS 版本 |
| `<mysql_port>` | 9004 | MySQL 兼容协议 |
| `<postgresql_port>` | 9005 | PostgreSQL 兼容协议 |

覆盖写法（`config.d/ports.xml`）：`<clickhouse>` 元素下直接写 `<http_port>18123</http_port>` 即可。三个坑：端口设为 `0` 会绑到随机临时端口，禁用某端口应该删掉配置项；只改端口不改 `listen_host`，服务可能仍在听 localhost；集群的 `interserver_http_port` 所有节点必须一致。没有冲突就别改默认端口。

### 用户与密码

`default` 用户默认无密码，第一件事把它锁上：

```sql
ALTER USER default IDENTIFIED BY 'YourStrongPassword';
```

之后客户端连接 `clickhouse-client --password`。更完整的用户管理见下文专门章节。

### 系统级调优

```bash
sudo sysctl -w vm.swappiness=1
echo 'vm.swappiness = 1' | sudo tee -a /etc/sysctl.conf
```

`vm.swappiness=1` 是最小化使用 Swap 而非禁用；要彻底关就 `swapoff -a`，代价是失去应急内存缓冲。

文件句柄：`/etc/security/limits.conf` 加 `clickhouse soft/hard nofile 262144`（tar/deb/rpm 部署都适用）；Docker 则走 `--ulimit`。

```bash
sudo sysctl -w vm.overcommit_memory=1   # 允许内存过度分配，适配大量 mmap
sudo sysctl -w vm.dirty_ratio=10        # 控制脏页比例，避免突发刷盘抖动
```

## 数据目录查看与迁移

### 确认数据在哪

```sql
-- 所有数据库的路径
SELECT name, engine, data_path FROM system.databases ORDER BY name;

-- 特定表
SELECT database, name, data_paths, metadata_path
FROM system.tables
WHERE database = 'your_db' AND name = 'your_table';
```

配置层面对应 `<path>` 标签，默认 `/var/lib/clickhouse/`。目录结构里最关心的几个：`data/` 传统引擎数据、`store/` Atomic 数据库按 UUID 组织的数据文件、`metadata/` 表定义、`tmp/` 查询与合并的临时文件、`access/` RBAC 权限数据。磁盘水位用 `df -hl` 看。

### 迁移到新目录（以 /data/clickhouse 为例）

```bash
# 1. 停服务
sudo systemctl stop clickhouse-server

# 2. 搬数据，注意源路径末尾的 /
sudo mkdir -p /data/clickhouse
sudo rsync -avz /var/lib/clickhouse/ /data/clickhouse/
```

斜杠是这段唯一的坑：`/var/lib/clickhouse/` 复制的是目录内容，少了斜杠会嵌套成 `/data/clickhouse/clickhouse/`。

```xml
<!-- 3. /etc/clickhouse-server/config.d/data_path.xml -->
<clickhouse>
    <path>/data/clickhouse/</path>
</clickhouse>
```

原配置里如有 `<tmp_path>`、`<user_files_path>`、`<backups_path>`，一并覆盖到新目录下。

```bash
# 4. 权限，别忘
sudo chown -R clickhouse:clickhouse /data/clickhouse/

# 5. 起服务验证
sudo systemctl start clickhouse-server
clickhouse-client --query "SELECT name, data_path FROM system.databases"
```

写一张测试表确认文件真的落在新目录，稳定运行一天后再清旧目录。也有偷懒的软链接方案（`mv` 过去再 `ln -s` 回来），对 ClickHouse 透明，但多一层间接，部分运维工具看不出真实路径，不太推荐。

## 用户与权限管理

官方推荐 SQL 驱动（RBAC），XML 方式留给没法登录的紧急场景。

### 创建用户与授权

```sql
-- 创建用户（默认即 sha256 存储）
CREATE USER IF NOT EXISTS api_writer IDENTIFIED BY 'StrongPass123';

-- 建号并顺手授权
CREATE USER report_user IDENTIFIED BY 'ReportPass789'
GRANTS SELECT ON my_database.*;

-- 授权
GRANT SELECT ON my_database.* TO report_user;          -- 只读
GRANT ALL ON my_database.* TO api_writer;              -- 库内全权
GRANT ACCESS MANAGEMENT ON *.* TO admin_user WITH GRANT OPTION;  -- 用户管理权
```

认证方式还有 `ldap`、`kerberos`、`no_password`（生产禁用）。登录来源可以收紧：`HOST LOCAL` 只许本地，`HOST IP '192.168.1.100'` 或 `'10.0.0.0/8'` 限定网段。

### 生效机制：不需要 FLUSH PRIVILEGES

ClickHouse 没有 MySQL 的 `FLUSH PRIVILEGES`。SQL 方式自动生效（集群下有几秒到一分钟的传播延迟，等就行）；只有手改 XML 才需要 `SYSTEM RELOAD CONFIG;` 或重启。

### 修改密码

```sql
ALTER USER api_writer IDENTIFIED BY 'NewStrongPassword!';

-- 集群务必带 ON CLUSTER
ALTER USER api_writer ON CLUSTER '{cluster}' IDENTIFIED WITH sha256_password BY 'NewStrongPassword!';

-- 改自己的
ALTER USER CURRENT_USER IDENTIFIED BY 'NewPassword';
```

忘记密码连不上时走 XML：编辑 `/etc/clickhouse-server/users.d/default-password.xml`，删掉密码配置重置为空，登进去再设。注意 SQL 设置的密码优先级高于 XML，SQL 还能用时改 XML 无效。

## 建库建表与 SQL 执行

### 建库建表

```sql
CREATE DATABASE IF NOT EXISTS gateway_logs;
```

默认引擎就是 `Atomic`，集群建库才需要 `ON CLUSTER`，且表引擎要换成 `ReplicatedMergeTree`，不能照搬单机建表语句。

一张典型的日志接入表——原始字段照日志格式落 String，物化列自动算出查询友好的类型：

```sql
CREATE TABLE IF NOT EXISTS gateway_logs.api_call_log
(
    -- 原始字段（与 APISIX log_format 一一对应）
    ts            String COMMENT '请求时间 unix 秒（$msec）',
    consumer_name LowCardinality(String) COMMENT '调用方（未认证为空）',
    route_id      LowCardinality(String) COMMENT 'APISIX 路由 ID',
    service_id    LowCardinality(String) COMMENT 'APISIX Service ID',
    uri           String COMMENT '请求路径含查询串',
    method        LowCardinality(String) COMMENT 'HTTP 方法',
    status        LowCardinality(String) COMMENT '响应状态码',
    client_ip     String COMMENT '客户端 IP',
    upstream      String COMMENT '上游地址',
    request_time  String COMMENT '总耗时秒',
    request_body  String COMMENT '请求体',

    -- 物化列：查询和报表直接用这些
    created_at    DateTime64(3, 'Asia/Shanghai') MATERIALIZED toDateTime64(toFloat64OrDefault(ts), 3, 'Asia/Shanghai') COMMENT '毫秒精度请求时间',
    status_code   UInt32 MATERIALIZED toUInt32OrDefault(status) COMMENT '数值状态码',
    latency_ms    Float64 MATERIALIZED toFloat64OrDefault(request_time) * 1000 COMMENT '毫秒耗时'
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, consumer_name)
TTL created_at + INTERVAL 90 DAY
SETTINGS index_granularity = 8192;
```

`index_granularity = 8192` 是 MergeTree 的稀疏索引粒度：调小索引更准但索引文件更大、写入更贵，调大反之。8192 是官方默认值，日志类表保持默认就好，除非压测证明查询延迟卡在这。

### 命令行执行 SQL

交互式就是 `clickhouse-client --user default --password` 进去粘贴，记得句尾分号加回车。

批量执行推荐写文件，最稳：

```bash
cat > /tmp/init_api_call_log.sql <<'EOF'
CREATE DATABASE IF NOT EXISTS gateway_logs;
CREATE TABLE IF NOT EXISTS gateway_logs.api_call_log (...同上...);
EOF

clickhouse-client --user default --password --multiquery < /tmp/init_api_call_log.sql
```

三个坑：非交互模式必须加 `--multiquery`，否则只执行第一条就报错；部分终端的 bracketed paste 会按行发送粘贴内容，逐行报语法错，改走文件；远程连接记得 `--host` / `--port 9000`。

验证：

```sql
SHOW TABLES FROM gateway_logs;
DESCRIBE TABLE gateway_logs.api_call_log;
```

## 集群部署概览

日增数据量或容灾要求超出单机时再上集群：

| 日增数据量 | 建议架构 |
|---|---|
| < 200 GB | 单机 MergeTree，做好备份与快照 |
| 200 GB ~ 2 TB | 2 分片 × 2 副本共 4 节点，`Distributed` 路由 + `ReplicatedMergeTree` 复制 |
| > 2 TB | 3+ 分片、每分片 2~3 副本，分片键兼顾写入均匀与分区裁剪 |

最容易漏的两件事：**9009 端口必须双向互通**，防火墙不放行副本就同步不了，Docker 部署还要 `-p 9009:9009`；**Keeper 独立部署**，别和 Server 混布，节点 4 GB 内存足够。建议所有节点的 `/etc/hosts` 提前固化主机名与 IP，DNS 不一致会导致节点识别错乱。具体的多节点配置以官方文档和 Docker Compose 模板为起点扩展。

## 常见问题速查

**启动失败 / 不监听端口**：前台跑一次看真实报错——`sudo -u clickhouse clickhouse-server --config-file=/etc/clickhouse-server/config.xml`。常见原因是配置语法错、端口占用、数据目录权限。

**远程连不上**：依次查 `listen_host` 是否放开、防火墙是否放行 8123/9000、用户 `HOST` 限制。

**Memory limit exceeded**：调 `users.d` 里的 `max_memory_usage`（单查询内存上限，物理内存的 15%~25%），按并发数反算；也别忘了看物理内存是不是本来就不够。

**副本不同步**：先确认 9009 双向可达，再查 Keeper 状态和各节点 DNS。

**/data 下出现 `clickhouse` 和 `clickhouse.0` 文件**：ClickHouse 自己不会生成这两个东西，基本是部署工具（多半是 Docker）在挂载时源路径不存在，自动建的占位文件。`file /data/clickhouse*` 看类型、`docker inspect <容器名>` 看 Mounts 确认，服务正常后删掉，然后把挂载目录提前 `mkdir -p` 修好。

**迁移后出现 clickhouse/clickhouse 嵌套**：`cp`/`rsync` 源路径末尾少了 `/`。停服务把内层内容提升一层、删嵌套空壳、修权限、重启。

## 附录

**数据目录迁移检查清单**：停服务 → `rsync -avz`（源路径末尾带斜杠）→ `config.d/data_path.xml` 写 `<path>` → `chown -R clickhouse:clickhouse` → 启动确认 active → SQL 确认 `data_path` → 写测试表确认落盘位置 → 稳定 24 小时再清旧目录。

**用户管理速查**：

| 操作 | SQL |
|---|---|
| 创建用户 | `CREATE USER u IDENTIFIED BY 'pwd';` |
| 建号并授权 | `CREATE USER u IDENTIFIED BY 'pwd' GRANTS SELECT ON db.*;` |
| 只读授权 | `GRANT SELECT ON db.* TO u;` |
| 库内全权 | `GRANT ALL ON db.* TO u;` |
| 改密码 | `ALTER USER u IDENTIFIED BY 'newpwd';` |
| 集群改密码 | `ALTER USER u ON CLUSTER '{cluster}' IDENTIFIED BY 'newpwd';` |
| 重载 XML 配置 | `SYSTEM RELOAD CONFIG;` |
| 当前用户 | `SELECT currentUser();` |

单机稳定运行之后，升级只是"下新压缩包、重跑一遍 `doinst.sh`"；配置变更全部走 `config.d/`、`users.d/`。把这两条习惯养住，后面怎么折腾都不慌。
