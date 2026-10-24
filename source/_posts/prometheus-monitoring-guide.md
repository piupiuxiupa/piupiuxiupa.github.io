---
title: Prometheus 监控保姆级教程：从单机到生产可用
date: 2026-10-24 23:55:37
tags: prometheus, monitoring, linux, grafana, alertmanager, devops
categories: Posts
excerpt: Prometheus 3.15 + node_exporter + Alertmanager + Grafana 完整落地指南：安装、抓取配置、服务发现、PromQL、12 条主机告警模板、告警路由与抑制、容量规划、高可用与安全加固。所有配置均经 promtool 校验与实机运行验证。
---

> 版本基线（2026-10 已核对官方 Releases）：
> Prometheus v3.15.1（2026-10-21）· Alertmanager v0.34.1（2026-09-17）· node_exporter v1.12.1（2026-07-14）· Grafana 取 apt.grafana.com 最新稳定版
> 适用场景：自建 Linux 服务器（物理机/虚机均可），二进制 + systemd 部署（不依赖 Docker/K8s）
> 所有 YAML/命令均为完整可运行片段；特定中间件（MySQL、Redis、Apache Doris 等）的接入不在本篇，另见对应接入指南。

## 目录

1. 快速上手：10 分钟最小可用
2. 核心概念（5 分钟看懂）
3. 安装 Prometheus Server（生产 systemd 方式）
4. prometheus.yml 配置详解与多种服务发现
5. node_exporter：Linux 主机监控
6. 其他常用 exporter 一览
7. PromQL 语法与常用查询速成
8. 告警规则：拿来即用模板（主机 12 条）
9. Alertmanager：路由、分组、抑制、静默、通知渠道
10. Grafana：数据源、Dashboard、变量
11. 容量规划、性能与存储调优
12. 高可用：双副本、Thanos/VictoriaMetrics、联邦
13. 安全加固（内网、TLS、Basic Auth、防火墙）
14. 日常运维命令速查
15. 上线检查清单
16. 参考资料

---

## 1. 快速上手：10 分钟最小可用

先跑起来再理解。在一台 Linux 机器上：

```bash
# 1) 下载解压（版本以 releases 页为准，此处为 2026-10 最新稳定版）
curl -LO https://github.com/prometheus/prometheus/releases/download/v3.15.1/prometheus-3.15.1.linux-amd64.tar.gz
tar xzf prometheus-3.15.1.linux-amd64.tar.gz
cd prometheus-3.15.1.linux-amd64

# 2) 写一个最小配置：抓自己
cat > /tmp/prom-min.yml <<'EOF'
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
EOF

# 3) 前台启动
./prometheus --config.file=/tmp/prom-min.yml --storage.tsdb.path=/tmp/prom-data

# 4) 浏览器打开 http://<本机IP>:9090
#    Status → Targets 看 localhost:9090 是否 UP
#    Graph 页输入 up 点 Execute，应返回 1
```

看到 `up 1` 就说明采集链路通了。后面的章节把这套东西做成生产级的：systemd 托管、抓 Linux 主机、告警发到邮箱/IM。

---

## 2. 核心概念（5 分钟看懂）

Prometheus 是"拉模型"的时序监控系统：

```
 ┌────────────┐  定时 GET /metrics   ┌──────────────────┐
 │ Prometheus │ ───────────────────▶ │  Exporter/应用    │  (node_exporter:9100
 │   Server   │ ◀─── 文本指标数据 ─── │  (暴露 /metrics)  │   应用自带 /metrics)
 └─────┬──────┘                      └──────────────────┘
       │ 规则评估(表达式)                       ▲
       ▼                                       │ 主动推送(仅少量场景)
 ┌────────────┐  告警通知            ┌──────────┴───────┐
 │Alertmanager│ ───────────────────▶ │ Pushgateway      │ (短暂任务用，慎用)
 │ (去重/路由) │      邮件/webhook/IM └──────────────────┘
 └────────────┘
       │ Prometheus 作为数据源
       ▼
 ┌────────────┐
 │  Grafana   │  可视化面板
 └────────────┘
```

要点：

- **时序（time series）**：一条时序 = 指标名 + 一组 label，如 `node_cpu_seconds_total{cpu="0",mode="idle",instance="10.0.0.11:9100"}`。label 组合每多一种，就多一条时序（这就是"基数 cardinality"问题的来源）。
- **四种指标类型**：
  - Counter：只增不减的累计值（请求数、字节数）。必须用 `rate()` 取斜率才有意义。
  - Gauge：可增可减的瞬时值（内存占用、连接数、温度）。
  - Histogram：分桶统计，配合 `histogram_quantile()` 算分位数。
  - Summary：客户端直接算好的分位数（带 `quantile` label），查询时直接按 label 取值。
- **抓取（scrape）**：Server 每隔 `scrape_interval` 主动 GET 目标的 `/metrics`，拿到形如 `metric_name{label} value` 的文本，存入本地 TSDB。
- **告警分两级**：Prometheus 里写"告警规则"（表达式持续为真 N 分钟 → 产生 alert），Alertmanager 负责"通知"（去重、分组、静默、路由到邮箱/IM）。职责分离，勿混淆。

---

## 3. 安装 Prometheus Server（生产 systemd 方式）

### 3.1 目录与用户规划

```bash
# 专用系统用户，无登录 shell
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus

sudo mkdir -p /etc/prometheus/rules /etc/prometheus/targets/nodes \
              /var/lib/prometheus /var/log/prometheus /opt/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus /var/log/prometheus
```

### 3.2 安装二进制

```bash
VER=3.15.1
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v${VER}/prometheus-${VER}.linux-amd64.tar.gz
tar xzf prometheus-${VER}.linux-amd64.tar.gz
sudo cp prometheus-${VER}.linux-amd64/prometheus  /usr/local/bin/
sudo cp prometheus-${VER}.linux-amd64/promtool    /usr/local/bin/
prometheus --version   # 确认版本
```

### 3.3 systemd 服务单元

`/etc/systemd/system/prometheus.service`：

```ini
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=30d \
  --storage.tsdb.retention.size=80GB \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle \
  --web.enable-admin-api
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
# 加固：只读文件系统 + 独立临时目录
ProtectSystem=strict
ReadOnlyPaths=/etc/prometheus
ReadWritePaths=/var/lib/prometheus
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

参数说明：

| 参数 | 作用 |
|---|---|
| `--storage.tsdb.retention.time=30d` | 数据保留时长，按磁盘预算调（见第 11 章） |
| `--storage.tsdb.retention.size=80GB` | 磁盘用量上限兜底，两者先到先删 |
| `--web.enable-lifecycle` | 允许 `POST /-/reload` 热加载配置（配合 nginx 限源使用） |
| `--web.enable-admin-api` | 允许 TSDB 快照/删除 API（做备份用，不用可去掉） |

### 3.4 启动与验证

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
curl -s http://127.0.0.1:9090/-/healthy    # 期望 Prometheus Server is Healthy.
```

### 3.5 版本升级流程

1. 下载新版二进制 → `promtool check config /etc/prometheus/prometheus.yml` 校验配置；
2. 读一遍目标版本的 CHANGELOG（3.x 大版本间有行为变更，如 3.0 重写了 UI、默认 Remote Write 2.0）；
3. `systemctl stop prometheus` → 替换 `/usr/local/bin/prometheus` → `start`；
4. 看 `/targets`、`/tsdb-status` 确认抓取恢复、旧数据仍可查（同 major 版本数据兼容）。

---

## 4. prometheus.yml 配置详解与多种服务发现

完整骨架 `/etc/prometheus/prometheus.yml`：

```yaml
global:
  scrape_interval: 15s        # 默认抓取间隔
  scrape_timeout: 10s         # 单次抓取超时，必须 < scrape_interval
  evaluation_interval: 15s    # 告警/记录规则评估间隔

# 告警与记录规则文件，支持通配
rule_files:
  - /etc/prometheus/rules/*.yml

# 告警发往哪里（Alertmanager，见第 9 章）
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['127.0.0.1:9093']

scrape_configs:
  # Prometheus 自监控（必配，监控系统的监控系统）
  - job_name: prometheus
    static_configs:
      - targets: ['127.0.0.1:9090']
```

### 4.1 三种常用目标管理方式

**static_configs**：少量固定机器，最简单。

```yaml
  - job_name: node
    static_configs:
      - targets: ['10.0.0.11:9100', '10.0.0.12:9100']
        labels: {env: prod, region: bj}
```

**file_sd**（推荐）：目标变更不用改主配置/不用 reload，改 JSON 文件 30 秒内自动生效。

```yaml
  - job_name: node
    file_sd_configs:
      - files: ['/etc/prometheus/targets/nodes/*.json']
        refresh_interval: 30s
```

`/etc/prometheus/targets/nodes/prod.json`：

```json
[
  {
    "targets": ["10.0.0.11:9100", "10.0.0.12:9100", "10.0.0.13:9100"],
    "labels": { "env": "prod", "role": "app" }
  },
  {
    "targets": ["10.0.0.21:9100"],
    "labels": { "env": "prod", "role": "db" }
  }
]
```

**consul_sd / http_sd**：机器上百台或已有 CMDB 时用，原理相同（返回目标列表），不再展开。

### 4.2 两级 relabel：改目标标签 vs 过滤指标

```yaml
  - job_name: node
    file_sd_configs:
      - files: ['/etc/prometheus/targets/nodes/*.json']
    relabel_configs:
      # 采集前：把 instance 改成自定义主机名（示例）
      - source_labels: [__address__]
        regex: '10\.0\.0\.11:9100'
        target_label: hostname
        replacement: app-01
    metric_relabel_configs:
      # 采集后：丢弃 go 运行时指标，省存储（node_exporter 场景常用）
      - source_labels: [__name__]
        regex: 'go_.*'
        action: drop
```

经验：`relabel_configs` 处理"目标"（改 instance/加标签/删目标），`metric_relabel_configs` 处理"指标"（drop 掉不想要的高基数序列）。控制基数主要靠后者。

---

## 5. node_exporter：Linux 主机监控

### 5.1 安装（所有被监控机）

```bash
VER=1.12.1
sudo useradd --no-create-home --shell /usr/sbin/nologin node_exporter
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v${VER}/node_exporter-${VER}.linux-amd64.tar.gz
tar xzf node_exporter-${VER}.linux-amd64.tar.gz
sudo cp node_exporter-${VER}.linux-amd64/node_exporter /usr/local/bin/
sudo mkdir -p /var/lib/node_exporter
```

`/etc/systemd/system/node_exporter.service`：

```ini
[Unit]
Description=Prometheus Node Exporter
After=network-online.target

[Service]
Type=simple
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter \
  --web.listen-address=:9100 \
  --collector.systemd \
  --collector.textfile.directory=/var/lib/node_exporter
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

写完可用 `systemd-analyze verify node_exporter.service` 做静态校验，再 `daemon-reload && enable --now`。

- `--collector.systemd`：开启 systemd 服务状态采集（`node_systemd_unit_state`）。白名单参数在新旧版本间改名过（`--collector.systemd.unit-whitelist` → `--collector.systemd.unit-include`），以 `node_exporter --help | grep -i systemd` 实际输出为准。
- `--collector.textfile.directory`：自定义指标入口，向该目录写 `.prom` 文本文件即可被采集，例如定时任务输出磁盘坏道计数、证书剩余天数。

批量下发（几十台机器的笨办法但有效）：

```bash
for h in 10.0.0.11 10.0.0.12 10.0.0.13; do
  scp node_exporter-${VER}.linux-amd64.tar.gz node_exporter.service $h:/tmp/
  ssh $h 'tar xzf /tmp/node_exporter-*.tar.gz -C /tmp/ &&
          sudo cp /tmp/node_exporter-*/node_exporter /usr/local/bin/ &&
          sudo mv /tmp/node_exporter.service /etc/systemd/system/ &&
          sudo systemctl daemon-reload && sudo systemctl enable --now node_exporter'
done
```

### 5.2 关键指标速查

| 指标 | 含义 |
|---|---|
| `node_cpu_seconds_total{mode="idle"}` | CPU 各模式累计秒数（Counter，取 rate 算使用率） |
| `node_memory_MemTotal_bytes` / `node_memory_MemAvailable_bytes` | 内存总量 / 可用量 |
| `node_filesystem_size_bytes` / `avail_bytes` / `files` / `files_free` | 文件系统容量与 inode |
| `node_disk_read_bytes_total` / `written_bytes_total` / `io_time_seconds_total` | 磁盘读写与 IO 时间 |
| `node_network_receive_bytes_total` / `transmit_bytes_total` | 网卡收发字节 |
| `node_load1` / `node_load5` / `node_load15` | 负载 |
| `node_vmstat_oom_kill` | OOM kill 次数（需内核 vmstat 支持） |
| `node_timex_offset_seconds` / `node_timex_sync_status` | 时钟偏移与 NTP 同步状态（分布式元数据/一致性协议强相关） |
| `node_systemd_unit_state{state="failed"}` | systemd 服务失败状态 |

### 5.3 常用面板查询（可直接贴 Grafana）

```text
CPU 使用率 %      100 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100
内存使用率 %      (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
磁盘使用率 %      (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs"}) * 100
磁盘 IO 利用率 %  rate(node_disk_io_time_seconds_total[5m]) * 100
下载带宽 B/s      rate(node_network_receive_bytes_total{device!~"lo|veth.*|docker.*"}[5m])
24h 后磁盘满      predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs"}[6h], 24*3600) < 0
```

---

## 6. 其他常用 exporter 一览

| Exporter | 端口 | 用途 |
|---|---|---|
| blackbox_exporter | 9115 | 黑盒探活：ICMP/TCP/HTTP，监控"用户视角可用性"（如探业务端口、探内网 DNS） |
| process-exporter | 9256 | 按进程名采集 CPU/内存/FD，适合补进程级视图 |
| mysqld_exporter | 9104 | MySQL 实例监控 |
| redis/nginx/haproxy exporter | 各异 | 对应中间件 |
| Pushgateway | 9091 | 接收短生命周期任务主动推送的指标。仅用于批处理任务自报，不要当普通采集通道用 |

blackbox 典型玩法（监控机探目标机 TCP 端口）：

```yaml
  - job_name: blackbox-tcp
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets: ['10.0.0.11:22', '10.0.0.11:80']   # 示例：SSH 和 Web 端口，换成你的业务端口
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115
```

成功判定：`probe_success == 1`。

---

## 7. PromQL 语法与常用查询速成

### 7.1 数据类型与选择器

- 瞬时向量：`node_memory_MemTotal_bytes`（取最新一个样本）
- 区间向量：`rate(node_cpu_seconds_total[5m])`（取最近 5 分钟所有样本）
- label 匹配：`=` 相等、`!=` 不等、`=~` 正则、`!~` 正则取反

```text
up{job="node", env=~"prod|staging", instance!~".*:9115"}
```

### 7.2 核心函数

| 函数 | 用途 | 备注 |
|---|---|---|
| `rate(m[5m])` | Counter 每秒平均增速 | 自动处理 Counter 重置，99% 场景用它 |
| `irate(m[5m])` | 瞬时增速 | 只看最后两个点，仅适合放大图细节 |
| `increase(m[1h])` | 1 小时增量 | rate×窗口的等价物 |
| `avg_over_time / max_over_time / min_over_time` | 对 Gauge 做时间聚合 | 内存/连接数看峰值用 max_over_time |
| `histogram_quantile(0.99, sum by (le) (rate(x_bucket[5m])))` | Histogram 算 P99 | 必须 `by (le)`，le 保留 |
| `predict_linear(m[6h], 86400)` | 线性外推 | 磁盘容量预测经典用法 |
| `absent(m)` | 无数据告警 | 目标整个消失时 up 也没有的兜底 |
| `label_replace(v, "dst", "$1", "src", "(.*)")` | 改标签 | 报表场景用 |

### 7.3 五个常见坑

1. 窗口至少是抓取间隔的 2 倍：15s 采集就用 `[30s]` 以上，推荐 `[5m]`，否则容易算出空值或锯齿。
2. Counter 别直接画图：累计值永远上涨，先 `rate()`。
3. `histogram_quantile` 之前忘了 `sum by (le)`，P99 会按桶各自算再瞎平均。
4. label 基数失控：给指标加 `user_id`、`sql_hash`、`path` 这类高基数 label，几天就能把 TSDB 打爆。接入新 exporter 先看 `/api/v1/status/tsdb` 的基数排行。
5. `sum(m)` 混合了不同单位/语义的序列时毫无意义，先想清楚 by/without。

---

## 8. 告警规则：拿来即用模板（主机 12 条）

严重级别约定：P0 电话/IM 立即响应（业务受损）；P1 工作时间处理（容量/隐患）；P2 观察项（趋势/优化）。

`/etc/prometheus/rules/host-rules.yml`：

```yaml
{% raw %}
groups:
  - name: host-rules
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels: {severity: P0, team: infra}
        annotations:
          summary: '实例失联 {{ $labels.job }} {{ $labels.instance }}'
          description: '抓取失败持续 2 分钟，机器宕机/网络/exporter 挂了。'

      - alert: HostHighCPU
        expr: 100 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100 > 85
        for: 10m
        labels: {severity: P1, team: infra}
        annotations:
          summary: 'CPU 高 {{ $labels.instance }}'
          description: 'CPU 使用率 {{ $value | printf "%.1f" }}% 超过 85% 持续 10 分钟。'

      - alert: HostOutOfMemory
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90
        for: 5m
        labels: {severity: P1, team: infra}
        annotations:
          summary: '内存高 {{ $labels.instance }}'
          description: '内存使用率 {{ $value | printf "%.1f" }}%。'

      - alert: HostOutOfDiskSpace
        expr: (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs"}) * 100 > 90
        for: 10m
        labels: {severity: P1, team: infra}
        annotations:
          summary: '磁盘空间高 {{ $labels.instance }} {{ $labels.mountpoint }}'
          description: '使用率 {{ $value | printf "%.1f" }}%。'

      - alert: HostInodeExhausted
        expr: node_filesystem_files_free{fstype!~"tmpfs|overlay|squashfs"} / node_filesystem_files{fstype!~"tmpfs|overlay|squashfs"} * 100 < 10
        for: 10m
        labels: {severity: P1, team: infra}
        annotations:
          summary: 'inode 不足 {{ $labels.instance }} {{ $labels.mountpoint }}'
          description: '剩余 inode 低于 10%，小文件风暴前兆。'

      - alert: HostDiskWillFillIn24Hours
        expr: predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs"}[6h], 24*3600) < 0
        for: 30m
        labels: {severity: P1, team: infra}
        annotations:
          summary: '磁盘将满 {{ $labels.instance }} {{ $labels.mountpoint }}'
          description: '按当前增速 24 小时内写满。'

      - alert: HostDiskIOSaturation
        expr: rate(node_disk_io_time_seconds_total[5m]) * 100 > 90
        for: 10m
        labels: {severity: P2, team: infra}
        annotations:
          summary: '磁盘 IO 饱和 {{ $labels.instance }} {{ $labels.device }}'
          description: 'IO 利用率 {{ $value | printf "%.0f" }}%。'

      - alert: HostLoadSaturation
        expr: node_load5 / count without (cpu, mode) (node_cpu_seconds_total{mode="idle"}) > 2
        for: 15m
        labels: {severity: P2, team: infra}
        annotations:
          summary: '负载饱和 {{ $labels.instance }}'
          description: 'load5 超过核数 2 倍持续 15 分钟。'

      - alert: HostOomKilled
        expr: increase(node_vmstat_oom_kill[10m]) > 0
        labels: {severity: P1, team: infra}
        annotations:
          summary: '发生 OOM {{ $labels.instance }}'
          description: '10 分钟内 OOM kill {{ $value }} 次，dmesg 查进程。'

      - alert: HostSystemdServiceFailed
        expr: node_systemd_unit_state{state="failed"} == 1
        for: 5m
        labels: {severity: P1, team: infra}
        annotations:
          summary: '服务失败 {{ $labels.instance }} {{ $labels.systemd_unit }}'
          description: 'systemd 单元进入 failed 状态。'

      - alert: HostClockSkew
        expr: abs(node_timex_offset_seconds) > 0.05
        for: 10m
        labels: {severity: P1, team: infra}
        annotations:
          summary: '时钟偏移 {{ $labels.instance }}'
          description: '偏移 {{ $value | printf "%.3f" }}s。分布式元数据与副本协议对时钟敏感，尽快修复 NTP。'

      - alert: HostNtpSyncLost
        expr: node_timex_sync_status == 0
        for: 10m
        labels: {severity: P2, team: infra}
        annotations:
          summary: 'NTP 未同步 {{ $labels.instance }}'
{% endraw %}
```

校验规则语法：

```bash
promtool check rules /etc/prometheus/rules/*.yml
```

两点工程建议：

- `for` 不能省略思考：没有 `for` 的告警在毛刺上疯狂触发，等于狼来了。
- 告警必须指向"下一个动作"：annotation 里写清楚先看哪个 dashboard / 跑哪条命令，否则值班的人只会机械重启。

---

## 9. Alertmanager：路由、分组、抑制、静默、通知渠道

### 9.1 安装

```bash
VER=0.34.1
sudo useradd --no-create-home --shell /usr/sbin/nologin alertmanager
cd /tmp
curl -LO https://github.com/prometheus/alertmanager/releases/download/v${VER}/alertmanager-${VER}.linux-amd64.tar.gz
tar xzf alertmanager-${VER}.linux-amd64.tar.gz
sudo cp alertmanager-${VER}.linux-amd64/alertmanager /usr/local/bin/
sudo cp alertmanager-${VER}.linux-amd64/amtool       /usr/local/bin/
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
```

systemd 单元写法与 Prometheus 相同，ExecStart 换成：

```ini
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --web.listen-address=:9093 \
  --cluster.listen-address=0.0.0.0:9094
```

### 9.2 核心配置 `/etc/alertmanager/alertmanager.yml`

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:465'
  smtp_from: 'alert@example.com'
  smtp_auth_username: 'alert@example.com'
  smtp_auth_password: '授权码'
  smtp_require_tls: false

# 路由树：从根往下匹配，第一个命中的子路由生效（continue 默认 false）
route:
  receiver: ops-mail
  group_by: ['alertname', 'cluster']
  group_wait: 30s        # 等 30s 攒一批同组告警一起发
  group_interval: 5m     # 同组新告警最早 5m 后再发一条
  repeat_interval: 4h    # 未恢复的告警每 4h 重复提醒
  routes:
    - matchers: [ 'severity = "P0"' ]
      receiver: oncall-im        # P0 走 IM/电话通道
      repeat_interval: 30m
      continue: false
    - matchers: [ 'severity = "P2"' ]
      receiver: ops-mail-digest  # P2 只进邮件日报箱
      repeat_interval: 12h

receivers:
  - name: ops-mail
    email_configs:
      - to: 'ops@example.com'
  - name: ops-mail-digest
    email_configs:
      - to: 'ops-digest@example.com'
  - name: oncall-im
    webhook_configs:
      # 钉钉：部署 prometheus-webhook-dingtalk 后填其地址
      # 飞书：用 PrometheusAlert / FlashDuty 之类转换器，Alertmanager 无原生飞书
      - url: 'http://127.0.0.1:8060/dingtalk/ops/send'
        send_resolved: true
    email_configs:
      - to: 'oncall@example.com'

# 抑制：实例挂了就别再轰炸它衍生的一堆指标告警
inhibit_rules:
  - source_matchers: [ 'alertname = InstanceDown' ]
    target_matchers: [ 'severity =~ "P1|P2"' ]
    equal: ['instance']
```

- matchers 列表写法（0.22+ 语法）：`'label = "value"'`、`'label =~ "regex"'`。
- `send_resolved: true`：恢复时也通知，务必开。

### 9.3 企业微信（原生支持）示例

```yaml
  - name: wecom
    wechat_configs:
      - corp_id: 'ww_xxx'
        agent_id: '1000002'
        api_secret: 'xxx'
        to_user: '@all'
        send_resolved: true
```

### 9.4 静默与运维

```bash
amtool check-config /etc/alertmanager/alertmanager.yml
amtool --alertmanager.url=http://127.0.0.1:9093 alert query
# 例行维护静默 2 小时（UI 上 Silences 页也能点）
amtool --alertmanager.url=http://127.0.0.1:9093 silence add instance="10.0.0.11.*" -c "磁盘更换窗口" -d 2h
```

### 9.5 高可用

两台 Alertmanager 组成 gossip 集群，通知自动去重（同一告警只发一次）：

```ini
# 实例 A
--cluster.peer=10.0.0.30:9094
# 实例 B
--cluster.peer=10.0.0.31:9094
```

Prometheus 侧 `alerting.alertmanagers` 里把两个地址都写上。

---

## 10. Grafana：数据源、Dashboard、变量

### 10.1 安装（Debian/Ubuntu 官方源）

```bash
sudo apt-get install -y apt-transport-https software-properties-common
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://apt.grafana.com/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update && sudo apt-get install -y grafana
sudo systemctl enable --now grafana-server   # UI 在 :3000，默认 admin/admin，首次登录强制改密
```

### 10.2 数据源即代码（provisioning）

`/etc/grafana/provisioning/datasources/prom.yml`：

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://127.0.0.1:9090
    isDefault: true
    jsonData:
      timeInterval: 15s
```

`systemctl restart grafana-server` 后数据源即存在。

### 10.3 导入面板

- Dashboards → Import → 填面板 ID `1860`（Node Exporter Full，主机监控事实标准）；
- 按需导入所用中间件/应用的官方或社区模板（多数知名组件都有现成面板，从组件官方文档找入口最稳）；
- 导入后检查每行的 datasource 映射到你的 Prometheus。

### 10.4 自建面板的变量技巧

Dashboard Variables 里加一个 `instance` 变量，Query 填：

```text
label_values(up{job="node"}, instance)
```

面板查询里用 `$instance` 引用，一个面板服务所有机器；其他面板同理用 `env`/`role` 标签做变量，多套环境复用同一模板。

---

## 11. 容量规划、性能与存储调优

### 11.1 决定成本的三个量

1. **活跃序列数**（`prometheus_tsdb_head_series`）：内存的主要消耗者。经验值：每 100 万活跃序列预留 4–8GB 内存（含查询余量，必须以实测校准，看 `process_resident_memory_bytes`）。控制手段：scrape_interval 别激进、`metric_relabel_configs` drop 高基数指标、别给业务指标塞 ID 类 label。
2. **每秒采样数** ≈ 活跃序列数 / scrape_interval：磁盘占用由此决定。
3. **保留时长**：磁盘 ≈ 每秒采样数 × 保留秒数 × 1.5 字节（压缩后经验值）。

例：5 万活跃序列、15s 采集、保留 30 天 → 3333 样本/s × 2,592,000s × 1.5B ≈ 13GB，再给 WAL/索引/峰值留 1.5–2 倍余量，规划 30GB 盘即可。

### 11.2 调优参数

| 参数/手段 | 说明 |
|---|---|
| `--storage.tsdb.retention.time` | 一般 15–30d 够用；更久的历史走第 12 章的远端存储 |
| `--storage.tsdb.retention.size` | 磁盘兜底，建议设为盘的 70–80% |
| query 高峰 OOM | 限并发：反代层限制；或升配。排查慢查询：/api/v1/status/tsdb 看重查询 |
| `--web.enable-otlp-receiver` | Prometheus 3.x 原生接 OpenTelemetry 推送（需要时再开） |
| SSD | TSDB 建议 SSD，HDD 在大量查询时 WAL 追不上 |

### 11.3 远端存储选型一句话

- 小规模自建（几十台机器）：本地 TSDB + 30d 完全够，别过度设计。
- 需要长期历史/大集群：VictoriaMetrics（单机版部署最简单、资源省，`vmagent` 抓取后 remote_write 过去）或 Thanos（贴 Prometheus 生态，对象存储存历史）。Mimir 也是同类选项。三选一即可，别同时上。

---

## 12. 高可用：双副本、Thanos/VictoriaMetrics、联邦

### 12.1 推荐的最省心 HA：双 Prometheus + 集群 Alertmanager

- 两台监控机跑**完全相同配置**的 Prometheus，各自独立抓全部目标；
- Grafana 数据源指向其中一台（或前置一个 nginx 均衡）；
- Alertmanager 两实例组 gossip 集群做通知去重（见 9.5）；
- 两台 Prometheus 数据略有差异属正常，告警表达式两侧一致即可。

这没有"故障自动切换"的花活，但组件最少、行为最可预测，是绝大多数自建环境的最优解。

### 12.2 Thanos（长期存储 + 全局视图）

架构：每台 Prometheus 旁挂 sidecar，把块数据上传对象存储；Querier 聚合查询两副本；Compactor 在对象存储上做降采样与压缩。适合多套 Prometheus + 需要一年以上历史。部署面较大，中小规模优先 VictoriaMetrics。

### 12.3 联邦（federation）：层级化拉取

上级 Prometheus 定期拉下级的聚合结果，适合多机房汇聚总览：

```yaml
  - job_name: federate
    honor_labels: true
    metrics_path: /federate
    params:
      'match[]':
        - '{job="node"}'
    static_configs:
      - targets: ['sub-prom.bj.internal:9090']
```

只拉聚合后的少量序列，别用联邦做全量同步。

---

## 13. 安全加固

指标端点是信息泄露面：`/metrics` 会暴露内网拓扑、软件版本，中间件指标还可能带库名、实例名等业务信息。把它当"内网资产测绘数据"对待。

1. **网络边界**：Prometheus 9090、Alertmanager 9093、Grafana 3000、各 exporter 只对监控网段开放。用防火墙限源（示例，按你的表结构改）：

```bash
sudo nft add rule inet filter input tcp dport {9100,9090,9093,3000} ip saddr 10.0.20.0/24 accept comment "monitor-mgmt"
```

2. **Prometheus 原生 TLS + Basic Auth**（exporter 同样支持 `--web.config.file`）：

```bash
# 生成 bcrypt 密码哈希（需要 apache2-utils）
htpasswd -nBC 10 '' admin | tr -d ':\n' > /tmp/hash
```

`/etc/prometheus/web.yml`：

```yaml
tls_server_config:
  cert_file: /etc/prometheus/tls/server.crt
  key_file: /etc/prometheus/tls/server.key
basic_auth_users:
  admin: <粘贴上面生成的 bcrypt 哈希>
```

启动参数追加 `--web.config.file=/etc/prometheus/web.yml`。Grafana 数据源里配对应 TLS/认证。

3. **或者用反代收口**（组件只绑 127.0.0.1，出口统一走 nginx）：

```nginx
server {
    listen 443 ssl;
    server_name prom.internal.example.com;
    ssl_certificate     /etc/nginx/tls/prom.crt;
    ssl_certificate_key /etc/nginx/tls/prom.key;

    location / {
        allow 10.0.20.0/24;
        deny all;
        auth_basic "Prometheus";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

好处：nginx 访问日志天然成了审计日志（Prometheus 自身无审计）。

4. **告警外发通道**：webhook/邮件里的告警文本含内网 IP 与指标值，IM 群也是泄露面。P0 通道收窄接收人；对外 SaaS（如第三方值班平台）评估数据存储合规。
5. **API 写接口**：`--web.enable-admin-api`、`--web.enable-lifecycle` 打开后，`/-/reload`、TSDB 快照/删除接口都可被调用——务必置于认证与限源之后。
6. **最小权限采集**：exporter 用专用低权用户跑（第 3/5/9 章均如此）；不要让 Prometheus 的抓取凭据与其他系统共享。

---

## 14. 日常运维命令速查

```bash
# 配置/规则校验
promtool check config /etc/prometheus/prometheus.yml
promtool check rules  /etc/prometheus/rules/*.yml

# 热加载配置（需 --web.enable-lifecycle）
curl -X POST http://127.0.0.1:9090/-/reload
# 或者：killall -HUP prometheus

# 命令行查询
promtool query instant http://127.0.0.1:9090 'up{job="node"}'
promtool query range  http://127.0.0.1:9090 'node_load5' --start=$(date -d '-1h' +%s) --end=$(date +%s) --step=60

# HTTP API
curl -s 'http://127.0.0.1:9090/api/v1/query?query=up'
curl -s 'http://127.0.0.1:9090/api/v1/query_range?query=node_load5&start=-1h&end=now&step=60s'

# 状态排障页
# /targets                各目标最近抓取结果与错误
# /tsdb-status            序列基数、top 指标
# /api/v1/status/tsdb     API 版基数分析（找高基数元凶）
# /rules /alerts          规则加载与当前告警

# 数据备份（需 --web.enable-admin-api）
curl -s -XPOST http://127.0.0.1:9090/api/v1/admin/tsdb/snapshot | jq -r '.data.name'
# 产物在 /var/lib/prometheus/snapshots/<name>，rsync 走即可

# 临时清理某指标（误采的高基数指标，需 admin api）
curl -s -XPOST http://127.0.0.1:9090/api/v1/admin/tsdb/delete_series?match[]=go_gc_duration_seconds

# Alertmanager 排障
amtool --alertmanager.url=http://127.0.0.1:9093 alert query
amtool --alertmanager.url=http://127.0.0.1:9093 config routes --config.file=/etc/alertmanager/alertmanager.yml show
```

---

## 15. 上线检查清单

1. [ ] 监控机独立（或至少与被监控对象故障域隔离），磁盘/内存按第 11 章预估
2. [ ] Prometheus / Alertmanager / Grafana 均 systemd 托管、开机自启、`Restart=on-failure`
3. [ ] 全部主机装 node_exporter 并入 file_sd 目标文件；业务/中间件 exporter 按需接入
4. [ ] `promtool check config/rules` 通过；`/targets` 全绿
5. [ ] 导入 Grafana 面板（主机 1860 + 所用中间件官方模板），数据源 timeInterval 与采集间隔一致
6. [ ] 告警规则上线（先跑一周 P2-only 观察阈值，再放开 P0/P1 通知）
7. [ ] Alertmanager 路由/分组/抑制验证过：人为 `systemctl stop node_exporter` 触发一次 InstanceDown，确认通知到达且恢复通知也到
8. [ ] 值班通道（邮箱/IM webhook）端到端演练过一次
9. [ ] 防火墙限源 + TLS/BasicAuth（或 nginx 反代收口）配置完成
10. [ ] retention（time/size）按磁盘预算设置
11. [ ] TSDB 快照备份脚本 + 异机存放，恢复流程演练过一次
12. [ ] 监控自身纳入监控：另一实例（或同级双副本之一）抓取本机 Prometheus 的 `up{job="prometheus"}`
13. [ ] 文档化：目标清单在哪改（file_sd 目录）、告警阈值谁定、升级流程谁执行

---

## 16. 参考资料

- Prometheus 官方文档：https://prometheus.io/docs/introduction/overview/
- 配置参考：https://prometheus.io/docs/prometheus/latest/configuration/configuration/
- 告警规则/Alertmanager：https://prometheus.io/docs/alerting/latest/overview/
- PromQL：https://prometheus.io/docs/prometheus/latest/querying/basics/
- node_exporter 指南：https://prometheus.io/docs/guides/node-exporter/
- Grafana 文档：https://grafana.com/docs/
- prometheus-webhook-dingtalk：https://github.com/timonwong/prometheus-webhook-dingtalk
- VictoriaMetrics：https://victoriametrics.com/ ；Thanos：https://thanos.io/
