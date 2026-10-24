---
title: Apache Doris 监控接入：Prometheus + Grafana 实战
date: 2026-10-24 23:55:37
tags: doris, prometheus, monitoring, grafana, devops
categories: Posts
excerpt: Doris FE/BE 自带 Prometheus 格式指标，无需插件。本文给出抓取配置、FE/BE 关键指标清单、11 条告警规则（含 Compaction Score、元数据同步、BE 磁盘）、Compaction 堆积的五步排查路径，以及版本间指标命名差异的校对方法。
---

> 本篇是《Prometheus 监控保姆级教程 —— Linux 服务器通用版》的配套篇：主篇讲 Prometheus 本身怎么装怎么用，本篇只讲 Doris 怎么接上去。前置条件：已有一套能跑的 Prometheus（还没有就先看主篇第 1–3 章）。
> 指标名已对照 Doris 官方 4.x 指标文档核对；Doris 1.x → 4.x 命名有增改，接入时务必按第 4 节的命令用实际 /metrics 输出校对。

## 1. 端口与指标端点

Doris FE/BE 进程原生暴露 Prometheus 格式指标，不需要额外 exporter。

| 角色 | 默认端口 | 指标端点 |
|---|---|---|
| FE | http_port `8030` | `http://fe_host:8030/metrics` |
| BE | webserver_port `8040` | `http://be_host:8040/metrics` |
| BE（bvar 细分指标） | brpc_port `8060` | `http://be_host:8060/brpc_metrics` |
| Broker | 无 | 不提供 |

先人工核对（fe.conf / be.conf 里端口可能改过）：

```bash
curl -s http://fe1:8030/metrics | head -5
curl -s http://be1:8040/metrics | grep -E '^doris_be_(tablet|disks)' | head
```

## 2. 抓取配置

```yaml
  - job_name: doris_fe
    metrics_path: /metrics
    static_configs:
      - targets: ['10.0.0.21:8030', '10.0.0.22:8030', '10.0.0.23:8030']
        labels: {group: fe, cluster: doris-prod}

  - job_name: doris_be
    metrics_path: /metrics
    static_configs:
      - targets: ['10.0.0.11:8040', '10.0.0.12:8040', '10.0.0.13:8040', '10.0.0.14:8040']
        labels: {group: be, cluster: doris-prod}
```

机器多或经常扩缩容时，改用 file_sd（写法见主教程第 4 章），加节点就是改 JSON，30 秒自动生效。

关键注意点：**部分集群级指标（如 `doris_fe_max_tablet_compaction_score`）由 Master FE 汇报，非 Master 上可能缺失或为 0**。所以：

- FE 全部抓，不是只抓 Master；
- 查询这类指标一律用 `max(...)` 聚合，别直接画单节点曲线。

## 3. FE 关键指标

| 指标 | 类型 | 含义/用法 |
|---|---|---|
| `doris_fe_qps` / `doris_fe_rps` | Gauge | 当前 FE QPS / RPS |
| `doris_fe_query_err_rate` | Gauge | 每秒失败查询数（直接用，不用再 rate） |
| `doris_fe_query_latency_ms{quantile="0.99"}` | Summary | 查询延迟分位数，直接取 label，不要套 histogram_quantile |
| `doris_fe_max_tablet_compaction_score` | Gauge | 全集群最大 Compaction Score（P0，告警核心指标） |
| `doris_fe_connection_total` | Gauge | MySQL 端口当前连接数 |
| `doris_fe_max_journal_id` | Gauge | 元数据日志最大 ID；多 FE 差值大 = 元数据同步问题 |
| `doris_fe_edit_log{type="current_bytes"}` | Gauge | 元数据日志当前大小 |
| `doris_fe_report_queue_size` | Gauge | BE 上报任务队列长度，反映 Master FE 处理能力 |
| `doris_fe_job{job=...,state=...}` | Gauge | 各类任务（导入等）数量/状态 |
| `doris_fe_edit_log_clean{type="failed"}` | Counter | 元数据日志清理失败，出现即需人工介入 |

## 4. BE 关键指标与版本校对

| 指标 | 含义/用法 |
|---|---|
| `doris_be_tablet_cumulative_max_compaction_score` / `..._base_max_compaction_score` | BE 侧 cumulative/base Compaction Score |
| `doris_be_disks_avail_capacity` / `doris_be_disks_total_capacity` | 数据盘可用/总量（按 path 标签分盘） |
| `doris_be_memory_allocated_bytes` / `doris_be_memory_pool_bytes_total` | BE 内存（jemalloc / 分池） |
| `doris_be_process_fd_num_used` / `..._limit_soft` | 文件句柄使用/软限 |
| `doris_be_query_scan_bytes` / `doris_be_query_scan_rows` | 扫描量（Counter，rate 后看趋势） |
| `doris_be_streaming_load_current_processing` | 进行中的 Stream Load 数 |
| `doris_be_tablet_version_num_distribution` | tablet 版本数分布（倾斜/版本堆积排查） |

版本差异校对工作流（接入前必做）：

```bash
curl -s http://be1:8040/metrics | grep -oE '^doris_be_[a-z_0-9]+' | sort -u > be-metrics.txt
curl -s http://fe1:8030/metrics | grep -oE '^doris_fe_[a-z_0-9]+' | sort -u > fe-metrics.txt
```

以实际输出对照官方指标文档（见文末参考资料）逐个确认再写告警。Doris 1.x → 4.x 变化不小：老版本的 `doris_be_compaction_score` 拆分成了 cumulative/base/time_series/size_based 多个；也有部分指标不带 `doris_be_` 前缀。**写错的指标名不会报错，只会让规则永远不触发。**

## 5. Doris 实战 PromQL

```text
集群总 QPS                  sum(doris_fe_qps)
集群查询错误率              sum(doris_fe_query_err_rate)
查询 P99 延迟(ms)           max(doris_fe_query_latency_ms{quantile="0.99"})
最大 Compaction Score      max(doris_fe_max_tablet_compaction_score)
BE 数据盘使用率 %           (1 - doris_be_disks_avail_capacity / doris_be_disks_total_capacity) * 100
FE 元数据同步差距           max(doris_fe_max_journal_id) - min(doris_fe_max_journal_id)
BE 内存(Bytes)              doris_be_memory_allocated_bytes
连接数                      doris_fe_connection_total
```

## 6. 告警规则（11 条）

`/etc/prometheus/rules/doris-rules.yml`（指标名已对照官方文档；标了"经验值/示例阈值"的，请结合自己集群基线调整，先以 P2 级别观察一周再放开通知）：

```yaml
{% raw %}
groups:
  - name: doris-rules
    rules:
      - alert: DorisFEDown
        expr: up{job="doris_fe"} == 0
        for: 1m
        labels: {severity: P0, team: doris}
        annotations:
          summary: 'FE 失联 {{ $labels.instance }}'
          description: 'FE /metrics 抓取失败 1 分钟。FE 挂了整个集群不可查询，立即处理。'

      - alert: DorisBEDown
        expr: up{job="doris_be"} == 0
        for: 1m
        labels: {severity: P0, team: doris}
        annotations:
          summary: 'BE 失联 {{ $labels.instance }}'
          description: 'BE /metrics 抓取失败 1 分钟。tablet 副本将开始降级，立即处理。'

      - alert: DorisCompactionScoreWarning
        expr: max(doris_fe_max_tablet_compaction_score) > 500
        for: 10m
        labels: {severity: P1, team: doris}
        annotations:
          summary: 'Compaction Score 偏高（经验阈值）'
          description: '集群最大 compaction score {{ $value | printf "%.0f" }} > 500，导入过快或 compaction 滞后，查询/写放大风险。'

      - alert: DorisCompactionScoreCritical
        expr: max(doris_fe_max_tablet_compaction_score) > 1000
        for: 5m
        labels: {severity: P0, team: doris}
        annotations:
          summary: 'Compaction Score 严重过高（经验阈值）'
          description: 'score {{ $value | printf "%.0f" }} > 1000，版本堆积，限流导入并排查 BE compaction。'

      - alert: DorisQueryErrorRateHigh
        expr: sum(doris_fe_query_err_rate) > 1
        for: 5m
        labels: {severity: P1, team: doris}
        annotations:
          summary: '查询错误率上升'
          description: '每秒失败查询 {{ $value | printf "%.1f" }}。审计日志查具体报错。'

      - alert: DorisQueryP99Slow
        expr: doris_fe_query_latency_ms{quantile="0.99"} > 5000
        for: 10m
        labels: {severity: P2, team: doris}
        annotations:
          summary: '查询 P99 慢 {{ $labels.instance }}'
          description: 'P99 {{ $value | printf "%.0f" }}ms，超过 5s 基线（示例阈值）。'

      - alert: DorisFEMetadataLag
        expr: max(doris_fe_max_journal_id) - min(doris_fe_max_journal_id) > 1000
        for: 10m
        labels: {severity: P1, team: doris}
        annotations:
          summary: 'FE 元数据同步落后'
          description: '多 FE journal ID 差距 {{ $value | printf "%.0f" }}，Follower 回放慢或网络问题。'

      - alert: DorisEditLogTooBig
        expr: doris_fe_edit_log{type="current_bytes"} > 1073741824
        for: 30m
        labels: {severity: P2, team: doris}
        annotations:
          summary: 'EditLog 偏大 {{ $labels.instance }}'
          description: '当前 editlog {{ $value | humanize1024 }}B，关注 image 生成是否正常（示例阈值）。'

      - alert: DorisBEDiskUsageHigh
        expr: (1 - doris_be_disks_avail_capacity / doris_be_disks_total_capacity) * 100 > 85
        for: 10m
        labels: {severity: P1, team: doris}
        annotations:
          summary: 'BE 数据盘高 {{ $labels.instance }} {{ $labels.path }}'
          description: '使用率 {{ $value | printf "%.1f" }}%。Doris 磁盘满会直接拒绝写入。'

      - alert: DorisBEFdNearLimit
        expr: doris_be_process_fd_num_used / doris_be_process_fd_num_limit_soft > 0.9
        for: 5m
        labels: {severity: P1, team: doris}
        annotations:
          summary: 'BE 文件句柄将耗尽 {{ $labels.instance }}'
          description: '已达软限制 {{ $value | printf "%.0f" }}%。'

      - alert: DorisFEConnectionHigh
        expr: doris_fe_connection_total > 1024
        for: 5m
        labels: {severity: P2, team: doris}
        annotations:
          summary: 'FE 连接数高 {{ $labels.instance }}'
          description: '当前 {{ $value | printf "%.0f" }} 连接（示例阈值，按 qe_max_connection 的 80% 调整）。'
{% endraw %}
```

```bash
promtool check rules /etc/prometheus/rules/doris-rules.yml
curl -X POST http://127.0.0.1:9090/-/reload   # 热加载
```

配套建议：Alertmanager 里加一条抑制——BE 挂掉时抑制同实例的其他 Doris 告警，避免风暴（写法见主教程 9.2 的 inhibit_rules）。

## 7. Grafana 面板

- Doris 官方在文档《监控和报警》（monitor-alert）页面提供 Prometheus 配置样例和 Grafana Dashboard 模板 JSON，直接 Import 即可，覆盖 FE/BE 概览、查询延迟、导入、Compaction 等面板；
- Doris 所在主机层导入 Node Exporter Full（面板 ID 1860）；
- 多套集群时用第 2 节的 `cluster`/`group` 标签做 Dashboard 变量，一套模板管所有集群。

## 8. Compaction Score 高的排查路径

Compaction Score 是 Doris 最值得盯的单指标，但它只在**持续走高**时是问题（导入高峰短时上冲后回落属正常）。顺序排查：

1. 看趋势确认是持续堆积还是毛刺：`max(doris_fe_max_tablet_compaction_score)` 近几小时曲线；
2. 定位是哪台 BE：按节点拆 `doris_be_tablet_cumulative_max_compaction_score` / `..._base_...`，看是否集中在个别节点（往往对应磁盘热点或 tablet 倾斜）；
3. 看是不是导入突增：stream load / insert 相关速率指标（`doris_be_streaming_load_current_processing`、`doris_fe_job{job="load"}`）是否同步上扬；
4. 看主机层磁盘 IO：node_exporter 的 `node_disk_io_time_seconds_total`（IO util）与 await——IO 打满时 compaction 追不上写入；
5. 处置：先限流导入止血，再调整 compaction 并发/磁盘配比（be.conf 相关参数随版本文档调整），tablet 倾斜严重时考虑重分布。

## 9. 与主机监控联动

Doris 机器同样要挂 node_exporter（主教程第 5 章），以下三条主机告警对 Doris 尤其重要：

- **时钟偏移 / NTP 失步**（HostClockSkew / HostNtpSyncLost）：Doris 元数据与副本协议对时钟敏感；
- **数据盘水位与写满预测**（HostOutOfDiskSpace / HostDiskWillFillIn24Hours）：BE 满盘直接拒绝写入；
- **文件句柄**：与第 6 节 `DorisBEFdNearLimit` 互为印证。

## 10. 接入检查清单

1. [ ] `/targets` 里 doris_fe / doris_be 两个 job 全绿
2. [ ] FE 全部节点都在抓取列表里（不只 Master）
3. [ ] 指标名按实际 /metrics 输出逐个校对过（第 4 节工作流）
4. [ ] 官方 Grafana 模板导入成功，cluster/group 变量工作正常
5. [ ] 三条 P0 告警（FE Down / BE Down / Compaction Critical）做一次实战演练：人为停一个 BE，确认告警到达、恢复通知也到
6. [ ] 抑制规则生效：BE 停机期间同实例衍生告警被 suppressed
7. [ ] compaction / 元数据同步 / BE 磁盘三条告警的阈值记录在案，注明是经验值及调整日期

## 11. 参考资料

- Doris 监控指标（P0 级指标清单，中文）：https://doris.apache.org/zh-CN/docs/dev/admin-manual/maint-monitor/metrics/
- Doris 监控和报警（官方 Prometheus/Grafana 搭建步骤与 Dashboard 模板）：https://doris.apache.org/zh-CN/docs/dev/admin-manual/maint-monitor/monitor-alert/
- Prometheus 告警规则语法：https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- 配套主教程：《Prometheus 监控保姆级教程 —— Linux 服务器通用版》（~/prometheus-tutorial.md）
