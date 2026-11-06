---
title: Redis 7.x 生产配置模板：容器化部署下的持久化、内存与安全基线
date: 2026-11-06 13:58:42
tags: [redis, docker, 运维]
categories: Posts
excerpt: 一份可以直接抄走改密码用的 Redis 7 容器化生产基线：RDB+AOF 混合持久化、内存上限与淘汰策略、多线程 IO、日志轮转与安全加固，附启动验证清单和踩坑排查表。
---

Redis 裸奔在公网上被清库、被写计划任务挖矿的案例每年都有，事后翻配置，翻来覆去就是那几个口子：没设密码、`0.0.0.0` 加端口直接映射公网、没设 `maxmemory`。这篇把容器化部署下这些口子一次性堵上，给一份能直接落地的基线：强密码、RDB + AOF 双持久化、内存上限、Docker 日志轮转，最后附启动验证清单和一份排查表。Redis 基础概念在[之前的入门篇](/2026/06/23/redis-basic/)里写过，这篇只谈生产落地。

**上手前必改三处**：`requirepass`（32 位以上强随机密码）、`maxmemory`（按实际内存调整）、`io-threads`（按核数调整）。端口映射不要暴露公网。

## 关键取舍

| 项 | 选择 | 理由 |
|---|---|---|
| 持久化 | RDB + AOF 同开，`aof-use-rdb-preamble yes` | RDB 管紧凑备份和快速全量恢复，AOF 管秒级数据安全；混合模式下重启恢复兼得两者 |
| fsync | `appendfsync everysec` | 最多丢约 1 秒数据，换性能值得；`always` 对磁盘 IOPS 要求太高 |
| maxmemory | 内存限额的 60%~70% | 给 fork 的写时复制、连接缓冲和系统进程留空间，避免 OOM |
| 淘汰策略 | 纯缓存 `allkeys-lru`；部分 key 必须保留则 `volatile-lru` | 按业务数据特征选 |
| 惰性释放 | `lazyfree-*` 全开 | 删大 key、过期淘汰放后台，主线程少一顿卡顿 |
| 多线程 IO | `io-threads` 约为核数的一半到 3/4 | 只加速网络读写，命令执行仍是单线程 |
| 日志 | `logfile ""` 输出 stdout，交给 Docker 落盘轮转 | Redis 原生做不到日志既进 `docker logs` 又进文件，二选一 |

## 完整配置（容器版）

保存为 `/opt/redis/redis.conf`。容器内约定：配置挂载到 `/usr/local/etc/redis/redis.conf`，数据目录挂载到 `/data`。

```conf
########################################
# 基础网络与进程
########################################
# 容器内监听所有接口，宿主机侧用端口映射收紧来源
bind 0.0.0.0
port 6379

# 容器网络已隔离 + 强密码，可关闭保护模式
protected-mode no

# 必须替换为 32 位以上强随机密码
requirepass Your_Strong_Random_Password_Here_32chars+

# 容器中必须前台运行，否则主进程退出，Docker 会认为容器挂了
daemonize no
supervised no

pidfile /var/run/redis_6379.pid

# 输出到标准输出，供 docker logs 捕获
loglevel notice
logfile ""

# 高并发下设 300，避免死连接堆积
timeout 300
tcp-keepalive 60
tcp-backlog 1024

# 与容器 nofile 上限匹配，见文末排查表
maxclients 10000

databases 16
always-show-logo no

########################################
# RDB 持久化
########################################
save 900 1
save 300 10
save 60 10000

stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
rdb-save-incremental-fsync yes
rdb-del-sync-files no

dbfilename dump.rdb

# 必须挂载到宿主机，且 UID 999 可写
dir /data

########################################
# AOF 持久化 + 混合持久化
########################################
appendonly yes
appendfilename "appendonly.aof"
appenddirname "appendonlydir"

appendfsync everysec

# AOF 重写以 RDB 格式开头，恢复快、完整性也保住
aof-use-rdb-preamble yes

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 重写期间不阻塞 fsync，写入密集场景保持 no
no-appendfsync-on-rewrite no

# 宕机导致 AOF 尾部截断时允许加载，避免起不来
aof-load-truncated yes

aof-rewrite-incremental-fsync yes

# 给 AOF 加时间戳注释，排查问题用得上
aof-timestamp-enabled yes

########################################
# 内存上限与淘汰策略
########################################
# 参照物是容器的 memory limit（docker --memory），不是宿主机总内存
# 示例：容器限额 16g，设 11g 左右
maxmemory 11gb
maxmemory-policy allkeys-lru
maxmemory-samples 10

########################################
# 惰性释放
########################################
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
lazyfree-lazy-user-del yes
lazyfree-lazy-user-flush yes
replica-lazy-flush yes

########################################
# 多线程 I/O（Redis 6+）
########################################
# 约为核数的 1/2～3/4，不要超过核数
io-threads 4
# 读多线程目前收益很小，官方文档明确不推荐，保持默认 no
io-threads-do-reads no

########################################
# 慢查询与延迟监控
########################################
slowlog-log-slower-than 10000
slowlog-max-len 256
latency-monitor-threshold 100

########################################
# 哈希与事件循环
########################################
activerehashing yes
hz 10
dynamic-hz yes

# 启动时尝试禁用透明大页（THP），宿主机层面关掉更彻底
disable-thp yes

########################################
# 安全加固（按需启用）
########################################
# 至少禁 FLUSHALL / FLUSHDB / KEYS
# 禁 CONFIG 会影响在线排查，依赖 CONFIG 运维的话保留或用 ACL 限制
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command KEYS ""
# rename-command CONFIG ""
# rename-command SHUTDOWN ""
# rename-command DEBUG ""

# Redis 7 的三道运行时保险丝，默认即 no，显式写出防手改
enable-protected-configs no
enable-debug-command no
enable-module-command no
```

## 宿主机部署的差异

同一份配置放到宿主机（systemd 管理）要改这些：

| 配置项 | 容器版 | 宿主机版 |
|---|---|---|
| `bind` | `0.0.0.0` | `127.0.0.1 内网IP`，不要写 `0.0.0.0` |
| `protected-mode` | `no` | `yes` |
| `daemonize` / `supervised` | `no` / `no` | `no` / systemd 管理可设 `systemd`（编译无 libsystemd 就用 `no`） |
| `logfile` | `""` | `/var/log/redis/redis-server.log` |
| `dir` | `/data` | `/var/lib/redis` |

日志里若出现 `systemd supervision requested or auto-detected, but Redis is compiled without libsystemd support!`，容器里把 `supervised` 设回 `no` 就好；宿主机上交给 systemd unit 管进程，不影响运行。

## 部署步骤

### 目录与权限

官方镜像默认以 `redis` 用户（UID 999）运行，挂载进来的文件要让它可读，数据目录要可写：

```bash
mkdir -p /opt/redis/data
sudo chown 999:999 /opt/redis/redis.conf
sudo chmod 644 /opt/redis/redis.conf
sudo chown -R 999:999 /opt/redis/data
```

宿主机开了 SELinux（CentOS/RHEL/Fedora）的话，挂载加 `:z` 或 `:Z` 标签。

### docker run

```bash
docker run -d \
  --name redis-prod \
  --memory 16g \
  --ulimit nofile=65535:65535 \
  -p 127.0.0.1:6379:6379 \
  -v /opt/redis/redis.conf:/usr/local/etc/redis/redis.conf:ro,z \
  -v /opt/redis/data:/data:z \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  redis:7.2.12 \
  redis-server /usr/local/etc/redis/redis.conf
```

几个细节：`-p 127.0.0.1:6379:6379` 只绑回环，需要内网访问就改成内网 IP；`--memory` 和配置里的 `maxmemory` 是一套规划，先定限额再算 `maxmemory`；`--ulimit` 直接给容器设 nofile 上限，不依赖宿主机 limits.conf（原因见系统层配套一节）。

### docker-compose

```yaml
services:
  redis:
    image: redis:7.2.12
    container_name: redis-prod
    user: "999:999"
    mem_limit: 16g
    ports:
      - "127.0.0.1:6379:6379"
    ulimits:
      nofile:
        soft: 65535
        hard: 65535
    volumes:
      - ./redis.conf:/usr/local/etc/redis/redis.conf:ro,z
      - ./data:/data:z
    command: redis-server /usr/local/etc/redis/redis.conf
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    restart: unless-stopped
```

## 系统层配套

以下都在**宿主机**执行，容器内替代不了。

### 内存 overcommit

```bash
sysctl vm.overcommit_memory=1
echo "vm.overcommit_memory = 1" >> /etc/sysctl.conf
```

解决启动日志的 `WARNING Memory overcommit must be enabled!`，否则低内存时后台保存或复制可能失败。

### 网络 backlog

```bash
sysctl net.core.somaxconn=1024
sysctl net.ipv4.tcp_max_syn_backlog=1024
```

同样持久化到 `/etc/sysctl.conf`，与配置里的 `tcp-backlog 1024` 对齐。

### 关闭透明大页 THP

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

配置里的 `disable-thp yes` 只是 Redis 启动时尝试设置，宿主机层面关掉更彻底，能明显减少延迟抖动。

### 文件描述符

```conf
# /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535
```

注意：limits.conf 只对 PAM 登录的进程生效，**容器内的 nofile 上限继承自 dockerd**，改这个文件对已运行的容器无效。要么像上文那样在 run/compose 里显式 `--ulimit`，要么调 dockerd 的 `LimitNOFILE`。`maxclients` 要留在这个上限之内。

## 启动后验证什么

```bash
docker ps -a --filter "name=redis-prod"
docker logs --tail 100 redis-prod
```

看到启动 Logo 和 `Ready to accept connections` 即正常。

连接与密码：

```bash
REDISCLI_AUTH='Your_Strong_Random_Password_Here_32chars+' \
redis-cli -h 127.0.0.1 -p 6379 PING
```

返回 `PONG` 就通。接着确认持久化状态：

```bash
REDISCLI_AUTH='密码' redis-cli INFO persistence
```

关注四个字段：`aof_enabled:1`、`rdb_last_bgsave_status:ok`、`aof_last_write_status:ok`、`aof_last_bgrewrite_status:ok`。

RDB 实际落盘路径用 `CONFIG GET dir` + `CONFIG GET dbfilename` 查，容器里应为 `/data/dump.rdb`。手动触发一次：

```bash
REDISCLI_AUTH='密码' redis-cli BGSAVE
REDISCLI_AUTH='密码' redis-cli INFO persistence
```

`rdb_last_bgsave_status` 为 `ok` 且 `rdb_last_save_time` 更新即成功；若是 `err`，查 `/data` 权限、磁盘空间、fork 内存。

Docker 日志落盘路径：

{% raw %}
```bash
docker inspect --format='{{.LogPath}}' redis-prod
```
{% endraw %}

这是 Docker 管理的 JSON 日志文件，可以直接 `tail`、`grep` 做归档分析。

## RDB 与 AOF：共存与"看不到 dump.rdb"

两者可以同开，本文就是这么配的。重启时 Redis **优先加载 AOF**（数据通常更完整），RDB 承担紧凑备份和快速全量恢复。另外后台 fork 任务同一时间只跑一个：AOF 重写进行中时，计划中的 RDB 保存会被推迟，反之亦然——这是正常行为，不是 bug。

看不到 `dump.rdb`，按这个顺序查：

1. **save 条件没触发**：`save 900 1` 这类规则没满足，或写入量本来就少。空闲实例长时间没有新 RDB 是正常的。
2. **被 AOF 重写推迟**：见上。
3. **文件在容器里没映射出来**：`CONFIG GET dir` 确认是不是 `/data`，`/data` 有没有挂到宿主机。
4. **保存失败**：手动 `BGSAVE` 后看 `INFO persistence` 的 `rdb_last_bgsave_status`，失败常见原因是目录权限、磁盘满、fork 内存不足。
5. **被覆盖**：容器停止时的保存会覆盖旧 `dump.rdb`，正常。

## 常见故障排查表

| 现象 | 原因 | 处理 |
|---|---|---|
| `can't open config file: Permission denied` | 配置文件属主与容器内 UID 999 不匹配 | `chown 999:999` + `chmod 644`，SELinux 加 `:z` |
| 容器启动即退、无日志 | `daemonize yes` 或 `supervised systemd` | 改 `daemonize no`、`supervised no` |
| `docker logs` 无输出但文件有日志 | `logfile` 指向了文件 | 改 `logfile ""`，走 Docker 日志 |
| 日志停在 `Configuration loaded` | 配置加载后进程退出：daemonize/supervised/dir 权限/端口冲突 | 前台交互跑一次 `redis-server` 看真实报错，重点查 `dir` 权限和端口 |
| `Memory overcommit must be enabled` 警告 | 宿主机没设 overcommit | `sysctl vm.overcommit_memory=1` |
| `systemd supervision requested...` 警告 | 容器里配了 supervised | 改 `supervised no` |
| 看不到 `dump.rdb` | save 未触发 / 目录未映射 / BGSAVE 失败 | `CONFIG GET dir` → `BGSAVE` → `INFO persistence` |
| 写操作报错 | RDB 保存失败 + `stop-writes-on-bgsave-error yes` | 查磁盘/权限/fork，修复后 `BGSAVE` |

## 安全加固清单

1. `requirepass` 用 32 位以上随机串，定期轮换。
2. 端口只绑 `127.0.0.1` 或内网 IP，云上再加安全组限制来源。
3. 危险命令至少禁 `FLUSHALL`、`FLUSHDB`、`KEYS`；`CONFIG` 禁不禁看运维方式，禁了就无法在线 `CONFIG GET/SET`。
4. 用 ACL 给不同应用建最小权限用户，别所有业务共用 default。
5. 容器不用 `--privileged`，不以 root 跑。
6. Docker 日志务必配 `max-size` / `max-file`，JSON 日志撑满磁盘这种事故没必要发生。
7. 定期备份 `/data/dump.rdb` 和 `/data/appendonlydir`，并且真的演练一次恢复。

改完密码、按内存设好 `maxmemory`，这份配置就可以直接上生产。真出问题时先看 `docker logs`，再看 `INFO persistence`，多数答案都在这两处。
