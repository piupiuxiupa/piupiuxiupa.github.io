---
title: Linux服务器日志收集完整方案（ELK Stack）
date: 2026-06-03 21:30:00
tags: linux, elk, logstash, elasticsearch, kibana, filebeat
categories: Posts
excerpt: 基于 Filebeat → Logstash → Elasticsearch → Kibana 的企业级日志收集架构部署方案
---

> 基于 **Filebeat → Logstash → Elasticsearch → Kibana (ELK Stack)** 的企业级日志收集架构

---

## 一、方案概述

### 1.1 架构图

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Filebeat   │───▶│  Logstash   │───▶│Elasticsearch│───▶│   Kibana    │
│  (采集端)    │    │  (处理端)    │    │  (存储+索引)  │    │  (可视化)    │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
      │                                      │
  多台服务器                               集群部署
```

### 1.2 方案优势

| 特性 | 说明 |
|------|------|
| **轻量采集** | Filebeat 资源占用极低（~30MB 内存） |
| **集中处理** | Logstash 统一解析、过滤、转换日志 |
| **全文搜索** | Elasticsearch 支持强大的全文检索能力 |
| **可视化** | Kibana 提供仪表盘、图表、告警 |
| **高可用** | 支持 Elasticsearch 集群和 Logstash 队列 |
| **灵活扩展** | 新增服务器只需部署 Filebeat |

### 1.3 日志类型覆盖

- 系统日志 (`/var/log/syslog`, `/var/log/messages`)
- 认证日志 (`/var/log/auth.log`, `/var/log/secure`)
- Nginx / Apache 访问日志 & 错误日志
- 应用日志（Java/Python/Go/Node.js 等）
- Docker 容器日志
- Cron 任务日志
- 内核日志 (`dmesg`)

---

## 二、环境要求

### 2.1 硬件要求

| 节点 | CPU | 内存 | 磁盘 | 数量 |
|------|-----|------|------|------|
| **Elasticsearch** | ≥ 4 核 | ≥ 8 GB | ≥ 100 GB SSD | ≥ 3 台（集群） |
| **Logstash** | ≥ 2 核 | ≥ 4 GB | ≥ 50 GB | ≥ 2 台 |
| **Kibana** | ≥ 2 核 | ≥ 2 GB | ≥ 20 GB | 1-2 台 |
| **Filebeat**（每台被采集服务器） | ≥ 0.5 核 | ≥ 0.5 GB | — | 视规模 |

### 2.2 软件要求

- 操作系统：CentOS 7+ / Ubuntu 18.04+ / Debian 10+
- Java：OpenJDK 17+（仅 Elasticsearch 和 Logstash 需要）
- 网络：采集端到中心端网络通畅

### 2.3 版本选择（推荐统一版本）

> **重要：ELK 组件版本必须保持一致，建议使用 9.x 最新稳定版（当前 9.4.2）**

- Elasticsearch: `9.4.x`
- Logstash: `9.4.x`
- Kibana: `9.4.x`
- Filebeat: `9.4.x`

---

## 三、部署步骤

### 3.1 Elasticsearch 部署

#### 3.1.1 安装

```bash
# 1. 导入 GPG Key
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

# 2. 添加 yum 源
cat > /etc/yum.repos.d/elasticsearch.repo << 'EOF'
[elasticsearch]
name=Elasticsearch repository
baseurl=https://artifacts.elastic.co/packages/9.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

# 3. 安装
yum install -y elasticsearch

# 4. Ubuntu/Debian 用户使用 apt
apt install -y apt-transport-https
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
echo "deb https://artifacts.elastic.co/packages/9.x/apt stable main" \
  > /etc/apt/sources.list.d/elastic-9.x.list
apt update && apt install -y elasticsearch
```

#### 3.1.2 配置

```bash
vim /etc/elasticsearch/elasticsearch.yml
```

```yaml
# ====== 基本配置 ======
cluster.name: my-log-cluster          # 集群名称，所有节点必须一致
node.name: node-1                     # 节点名称，每个节点唯一
path.data: /var/lib/elasticsearch     # 数据目录
path.logs: /var/log/elasticsearch     # 日志目录

# ====== 网络配置 ======
network.host: 0.0.0.0                 # 监听地址
http.port: 9200                       # HTTP 端口
transport.port: 9300                  # 节点通信端口

# ====== 集群发现 ======
discovery.seed_hosts: ["192.168.1.101", "192.168.1.102", "192.168.1.103"]
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]

# ====== 安全配置 ======
# 9.x 默认已启用安全功能（TLS + 认证）
# 安装时会自动生成证书和密码，记录好 elastic 用户的初始密码
# 首次启动后使用 elasticsearch-reset-password 工具重置密码
# elasticsearch-reset-password -u elastic

# ====== JVM 内存（不要超过物理内存的 50%） ======
```

```bash
# JVM 内存配置
vim /etc/elasticsearch/jvm.options
# 修改为：
# -Xms4g
# -Xmx4g
```

#### 3.1.3 启动与验证

```bash
# 启动
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch

# 验证（9.x 默认启用 HTTPS，需带认证）
curl -u elastic:your_password https://localhost:9200/_cluster/health?pretty
# 或忽略证书验证（测试环境）
curl -k -u elastic:your_password https://localhost:9200/_cluster/health?pretty
```

---

### 3.2 Logstash 部署

#### 3.2.1 安装

```bash
yum install -y logstash
# 或 apt install -y logstash
```

#### 3.2.2 配置管道

创建主配置文件：

```bash
vim /etc/logstash/conf.d/01-log-pipeline.conf
```

```ruby
# ========================================
# 输入 - 接收 Filebeat 发来的日志
# ========================================
input {
  beats {
    port => 5044
    host => "0.0.0.0"
  }
}

# ========================================
# 过滤 - 解析和处理日志
# ========================================

# --- Nginx 访问日志解析 ---
filter {
  if [tags] and "nginx-access" in [tags] {
    grok {
      match => { "message" => '%{IPORHOST:client_ip} - - \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{DATA:request_uri} HTTP/%{NUMBER:http_version}" %{NUMBER:status_code} %{NUMBER:body_bytes_sent} "%{DATA:referer}" "%{DATA:user_agent}"' }
    }
    date {
      match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
      target => "@timestamp"
    }
    geoip {
      source => "client_ip"
      target => "geoip"
    }
    mutate {
      remove_field => ["message"]
    }
  }

  # --- Nginx 错误日志解析 ---
  if [tags] and "nginx-error" in [tags] {
    grok {
      match => { "message" => '%{TIMESTAMP_ISO8601:timestamp} \[%{DATA:log_level}\] %{NUMBER:pid}#%{NUMBER:tid}: \*%{NUMBER:connection_id} %{GREEDYDATA:error_message}' }
    }
    date {
      match => ["timestamp", "YYYY-MM-DD HH:mm:ss"]
      target => "@timestamp"
    }
  }

  # --- 系统日志解析 ---
  if [tags] and "syslog" in [tags] {
    grok {
      match => { "message" => "%{SYSLOGLINE}" }
    }
    date {
      match => ["timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss"]
      target => "@timestamp"
    }
  }

  # --- 认证日志解析 ---
  if [tags] and "auth" in [tags] {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{HOSTNAME:host} %{DATA:process}\[%{NUMBER:pid}\]: %{GREEDYDATA:auth_message}" }
    }
  }

  # --- Docker 容器日志 ---
  if [tags] and "docker" in [tags] {
    json {
      source => "message"
      target => "docker"
    }
    date {
      match => ["docker.time", "ISO8601"]
      target => "@timestamp"
    }
  }

  # --- 通用应用日志（JSON 格式）---
  if [tags] and "application-json" in [tags] {
    json {
      source => "message"
    }
  }
}

# ========================================
# 输出 - 发送到 Elasticsearch
# ========================================
output {
  # 按日志类型写入不同索引
  if "nginx-access" in [tags] {
    elasticsearch {
      hosts => ["https://192.168.1.101:9200", "https://192.168.1.102:9200"]
      index => "nginx-access-%{+YYYY.MM.dd}"
      user => "elastic"
      password => "your_password"
      ssl_certificate_verification => false
    }
  } else if "nginx-error" in [tags] {
    elasticsearch {
      hosts => ["https://192.168.1.101:9200", "https://192.168.1.102:9200"]
      index => "nginx-error-%{+YYYY.MM.dd}"
      user => "elastic"
      password => "your_password"
      ssl_certificate_verification => false
    }
  } else if "syslog" in [tags] {
    elasticsearch {
      hosts => ["https://192.168.1.101:9200", "https://192.168.1.102:9200"]
      index => "syslog-%{+YYYY.MM.dd}"
      user => "elastic"
      password => "your_password"
      ssl_certificate_verification => false
    }
  } else if "auth" in [tags] {
    elasticsearch {
      hosts => ["https://192.168.1.101:9200", "https://192.168.1.102:9200"]
      index => "auth-%{+YYYY.MM.dd}"
      user => "elastic"
      password => "your_password"
      ssl_certificate_verification => false
    }
  } else {
    elasticsearch {
      hosts => ["https://192.168.1.101:9200", "https://192.168.1.102:9200"]
      index => "other-logs-%{+YYYY.MM.dd}"
      user => "elastic"
      password => "your_password"
      ssl_certificate_verification => false
    }
  }
}
```

#### 3.2.3 JVM 内存配置

```bash
vim /etc/logstash/jvm.options
# -Xms2g
# -Xmx2g
```

#### 3.2.4 启动与验证

```bash
systemctl enable logstash
systemctl start logstash

# 测试配置文件语法
/usr/share/logstash/bin/logstash --path.settings /etc/logstash \
  -f /etc/logstash/conf.d/01-log-pipeline.conf --config.test_and_exit

# 查看日志
tail -f /var/log/logstash/logstash-plain.log
```

---

### 3.3 Kibana 部署

#### 3.3.1 安装

```bash
yum install -y kibana
# 或 apt install -y kibana
```

#### 3.3.2 配置

```bash
vim /etc/kibana/kibana.yml
```

```yaml
server.port: 5601
server.host: "0.0.0.0"
server.name: "kibana-01"

elasticsearch.hosts: ["https://192.168.1.101:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "your_password"
elasticsearch.ssl.verificationMode: none

# 中文界面
i18n.locale: "zh-CN"

# 日志输出
logging.dest: /var/log/kibana/kibana.log
```

#### 3.3.3 启动

```bash
systemctl enable kibana
systemctl start kibana
```

#### 3.3.4 Nginx 反向代理（推荐）

```nginx
server {
    listen 80;
    server_name kibana.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:5601;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

### 3.4 Filebeat 部署（采集端）

#### 3.4.1 安装

```bash
# 在每台需要采集日志的服务器上执行
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

cat > /etc/yum.repos.d/elastic.repo << 'EOF'
[elasticsearch]
name=Elasticsearch repository
baseurl=https://artifacts.elastic.co/packages/9.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

yum install -y filebeat
```

#### 3.4.2 配置

```bash
vim /etc/filebeat/filebeat.yml
```

```yaml
# ==============================
# Filebeat 配置
# ==============================

# ====== Kibana 连接（用于加载索引模板等）======
setup.kibana:
  host: "http://192.168.1.100:5601"

# ====== Elasticsearch 连接（用于加载模板）=====
setup.ilm.enabled: false

# ====== 模块加载 ======
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

# ====== 日志采集定义 ======
filebeat.inputs:

  # --- 系统日志 ---
  - type: log
    enabled: true
    paths:
      - /var/log/syslog          # Ubuntu/Debian
      - /var/log/messages         # CentOS/RHEL
    tags: ["syslog"]
    fields:
      host_name: "${HOSTNAME}"
      log_type: "syslog"
    fields_under_root: true

  # --- 认证日志 ---
  - type: log
    enabled: true
    paths:
      - /var/log/auth.log         # Ubuntu/Debian
      - /var/log/secure           # CentOS/RHEL
    tags: ["auth"]
    fields:
      host_name: "${HOSTNAME}"
      log_type: "auth"
    fields_under_root: true

  # --- Nginx 访问日志 ---
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
    tags: ["nginx-access"]
    fields:
      host_name: "${HOSTNAME}"
      log_type: "nginx-access"
    fields_under_root: true

  # --- Nginx 错误日志 ---
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/error.log
    tags: ["nginx-error"]
    fields:
      host_name: "${HOSTNAME}"
      log_type: "nginx-error"
    fields_under_root: true

  # --- Docker 容器日志 ---
  - type: log
    enabled: true
    paths:
      - /var/lib/docker/containers/*/*-json.log
    tags: ["docker"]
    json.keys_under_root: true
    fields:
      host_name: "${HOSTNAME}"
      log_type: "docker"
    fields_under_root: true

  # --- 应用日志（通用）---
  - type: log
    enabled: true
    paths:
      - /data/logs/app/*.log
      - /data/logs/app/**/*.log
    tags: ["application-json"]
    fields:
      host_name: "${HOSTNAME}"
      log_type: "application"
    fields_under_root: true

  # --- Cron 日志 ---
  - type: log
    enabled: true
    paths:
      - /var/log/cron
    tags: ["cron"]
    fields:
      host_name: "${HOSTNAME}"
      log_type: "cron"
    fields_under_root: true

# ====== 处理器 ======
processors:
  - add_cloud_metadata: ~
  - add_host_metadata:
      netinfo.enabled: true

# ====== 输出到 Logstash ======
output.logstash:
  hosts: ["192.168.1.200:5044"]
  loadbalance: true           # 多个 Logstash 时负载均衡
  worker: 2                   # 发送工作线程数
  bulk_max_size: 2048         # 批量发送大小

# ====== 日志文件 ======
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0640

# ====== 注册文件（记录已读位置）=====
path.data: /var/lib/filebeat
path.logs: /var/log/filebeat
```

#### 3.4.3 启用内置模块（可选）

```bash
# 启用 Nginx 模块
filebeat modules enable nginx

# 查看可用模块
filebeat modules list
```

模块配置文件位置：`/etc/filebeat/modules.d/nginx.yml`

```yaml
- module: nginx
  access:
    enabled: true
    var.paths: ["/var/log/nginx/access.log*"]
  error:
    enabled: true
    var.paths: ["/var/log/nginx/error.log*"]
```

#### 3.4.4 启动与验证

```bash
# 检查配置
filebeat test config -c /etc/filebeat/filebeat.yml

# 测试到 Logstash 的连接
filebeat test output -c /etc/filebeat/filebeat.yml

# 启动
systemctl enable filebeat
systemctl start filebeat

# 查看状态
systemctl status filebeat
```

---

## 四、批量部署脚本

### 4.1 Filebeat 一键部署脚本

在每台被采集服务器上执行：

```bash
#!/bin/bash
# deploy-filebeat.sh - Filebeat 自动化部署脚本
set -e

# ====== 配置变量 ======
LOGSTAST_SERVER="192.168.1.200:5044"
KIBANA_SERVER="http://192.168.1.100:5601"
HOSTNAME_VAL=$(hostname)

# ====== 安装 Filebeat ======
echo "[+] 安装 Filebeat..."
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

cat > /etc/yum.repos.d/elastic.repo << EOF
[elasticsearch]
name=Elasticsearch repository
baseurl=https://artifacts.elastic.co/packages/9.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

yum install -y filebeat

# ====== 生成配置 ======
echo "[+] 生成配置..."
cat > /etc/filebeat/filebeat.yml << EOF
setup.kibana:
  host: "${KIBANA_SERVER}"

filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/messages
      - /var/log/syslog
    tags: ["syslog"]
    fields:
      host_name: "${HOSTNAME_VAL}"
      log_type: "syslog"
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /var/log/secure
      - /var/log/auth.log
    tags: ["auth"]
    fields:
      host_name: "${HOSTNAME_VAL}"
      log_type: "auth"
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
    tags: ["nginx-access"]
    fields:
      host_name: "${HOSTNAME_VAL}"
      log_type: "nginx-access"
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /var/log/nginx/error.log
    tags: ["nginx-error"]
    fields:
      host_name: "${HOSTNAME_VAL}"
      log_type: "nginx-error"
    fields_under_root: true

processors:
  - add_cloud_metadata: ~
  - add_host_metadata: ~

output.logstash:
  hosts: ["${LOGSTAST_SERVER}"]
  loadbalance: true

logging.level: info
logging.to_files: true
EOF

# ====== 启动 ======
echo "[+] 启动 Filebeat..."
systemctl daemon-reload
systemctl enable filebeat
systemctl restart filebeat

echo "[+] Filebeat 部署完成！"
echo "[+] 服务器: ${HOSTNAME_VAL}"
```

### 4.2 使用 Ansible 批量部署

```yaml
---
- name: 部署 Filebeat 到所有服务器
  hosts: log_servers
  become: yes

  vars:
    logstash_host: "192.168.1.200:5044"
    kibana_host: "http://192.168.1.100:5601"
    filebeat_version: "9.4.2"

  tasks:
    - name: 添加 Elastic GPG Key
      rpm_key:
        state: present
        key: https://artifacts.elastic.co/GPG-KEY-elasticsearch

    - name: 添加 Elastic yum 仓库
      yum_repository:
        name: elasticsearch
        description: Elasticsearch repository
        baseurl: https://artifacts.elastic.co/packages/9.x/yum
        gpgcheck: yes
        gpgkey: https://artifacts.elastic.co/GPG-KEY-elasticsearch
        enabled: yes
        state: present

    - name: 安装 Filebeat
      yum:
        name: "filebeat-{{ filebeat_version }}"
        state: present

    - name: 生成配置文件
      template:
        src: templates/filebeat.yml.j2
        dest: /etc/filebeat/filebeat.yml
      notify: restart filebeat

    - name: 启动并启用 Filebeat
      systemd:
        name: filebeat
        state: started
        enabled: yes

  handlers:
    - name: restart filebeat
      systemd:
        name: filebeat
        state: restarted
```

---

## 五、Kibana 使用指南

### 5.1 创建索引模式

1. 登录 Kibana → **管理** → **Stack Management**
2. 进入 **Kibana** → **索引模式** → **创建索引模式**
3. 输入索引模式：`nginx-access-*`
4. 选择时间字段：`@timestamp`
5. 对每个日志类型重复上述步骤

### 5.2 创建仪表盘

**推荐仪表盘：**

| 仪表盘 | 内容 |
|--------|------|
| **Nginx 监控** | 请求量趋势、状态码分布、Top 10 URL、响应时间、地理位置 |
| **安全审计** | SSH 登录记录、sudo 操作、失败登录尝试 |
| **系统健康** | CPU/内存/磁盘事件、OOM Kill、内核告警 |
| **应用监控** | ERROR 数量趋势、异常堆栈、慢请求 |
| **综合概览** | 所有日志的汇总视图 |

### 5.3 配置告警

```json
{
  "name": "高频 5xx 错误告警",
  "condition": {
    "type": "threshold",
    "threshold": {
      "value": 100,
      "metric": "count",
      "comparison": "above"
    },
    "query": {
      "bool": {
        "filter": [
          { "range": { "@timestamp": { "gte": "now-5m" } } },
          { "terms": { "status_code": [500, 502, 503, 504] } }
        ]
      }
    }
  },
  "actions": [
    {
      "type": "webhook",
      "url": "https://your-webhook-url/alert"
    }
  ]
}
```

### 5.4 常用查询语句

```
# 查看某个 IP 的所有请求
client_ip: "1.2.3.4"

# 查找 5xx 错误
status_code: [500 TO 599]

# 查找失败的 SSH 登录
tags: "auth" AND auth_message: "Failed password"

# 查找某个时间段的高峰
@timestamp: ["2026-01-01T00:00:00" TO "2026-01-01T23:59:59"]

# 组合查询
tags: "nginx-access" AND status_code: 403 AND NOT client_ip: "10.0.0.0/8"
```

---

## 六、索引生命周期管理（ILM）

### 6.1 创建 ILM 策略

通过 Kibana Dev Tools 执行：

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "1d",
            "max_docs": 10000000
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### 6.2 创建索引模板

```json
PUT _index_template/nginx-access-template
{
  "index_patterns": ["nginx-access-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "nginx-access"
    }
  }
}
```

---

## 七、性能优化

### 7.1 Elasticsearch 优化

```yaml
# elasticsearch.yml
indices.memory.index_buffer_size: 30%
thread_pool.write.size: 4
thread_pool.search.size: 12
indices.fielddata.cache.size: 40%
indices.query.cache.size: 25%
```

### 7.2 Filebeat 优化

```yaml
# filebeat.yml
queue.mem:
  events: 4096
  flush.min_events: 512
  flush.timeout: 1s

output.logstash:
  worker: 4
  bulk_max_size: 4096
```

### 7.3 Logstash 优化

```ruby
# pipeline 配置
pipeline.workers: 4
pipeline.batch.size: 125
pipeline.batch.delay: 50

# 持久化队列（防丢失）
queue.type: persisted
path.queue: /var/lib/logstash/queue
queue.page_capacity: 250mb
```

---

## 八、安全加固

### 8.1 传输加密

```yaml
# Filebeat → Logstash SSL
output.logstash:
  hosts: ["logstash-server:5044"]
  ssl.enabled: true
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
  ssl.certificate: "/etc/filebeat/client.crt"
  ssl.key: "/etc/filebeat/client.key"
```

### 8.2 访问控制

```bash
# 创建专用用户
/usr/share/elasticsearch/bin/elasticsearch-users create \
  -r filebeat_writer \
  -p "StrongPassword123!" filebeat

/usr/share/elasticsearch/bin/elasticsearch-users create \
  -r kibana_reader \
  -p "StrongPassword123!" kibana_user
```

### 8.3 防火墙规则

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="5044" protocol="tcp" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="9200" protocol="tcp" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="5601" protocol="tcp" accept'
firewall-cmd --reload
```

---

## 九、运维监控

### 9.1 Filebeat 健康检查

```bash
#!/bin/bash
FILEBEAT_STATUS=$(systemctl is-active filebeat)
LOGSTASH_PING=$(timeout 5 bash -c 'echo > /dev/tcp/192.168.1.200/5044' 2>&1 && echo "UP" || echo "DOWN")

if [ "$FILEBEAT_STATUS" != "active" ] || [ "$LOGSTASH_PING" = "DOWN" ]; then
    echo "[ALERT] Filebeat 状态: $FILEBEAT_STATUS | Logstash: $LOGSTASH_PING"
    systemctl restart filebeat
fi

# 添加到 crontab
# */5 * * * * /opt/scripts/check-filebeat.sh >> /var/log/filebeat-check.log 2>&1
```

### 9.2 Elasticsearch 集群监控 API

```bash
# 9.x 默认 HTTPS + 认证
ES_URL="https://localhost:9200"
ES_AUTH="-u elastic:your_password -k"

# 集群健康
curl -s $ES_AUTH "$ES_URL/_cluster/health?pretty"

# 节点状态
curl -s $ES_AUTH "$ES_URL/_cat/nodes?v"

# 索引大小
curl -s $ES_AUTH "$ES_URL/_cat/indices?v&s=index"
```

---

## 十、故障排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Filebeat 不发送日志 | 注册文件损坏 | 删除 `/var/lib/filebeat/registry/` 重新注册 |
| Logstash 内存溢出 | Grok 正则过于复杂 | 优化正则表达式，增加内存 |
| Elasticsearch 索引只读 | 磁盘超过水位线 | 清理旧索引，调整水位线 |
| Kibana 无法连接 ES | ES 集群未就绪 | 先恢复 ES 集群健康 |
| 日志丢失 | 队列满或重启 | 启用持久化队列，增加队列容量 |

```bash
# 各组件日志查看
journalctl -u filebeat -f
tail -f /var/log/logstash/logstash-plain.log
tail -f /var/log/elasticsearch/my-log-cluster.log
tail -f /var/log/kibana/kibana.log
```

---

## 十一、备选方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **ELK Stack** | 功能全面、生态成熟 | 资源占用大 | 中大型企业 |
| **EFK (Fluentd)** | 轻量、插件丰富 | 性能不如 ELK | Kubernetes 环境 |
| **Loki + Promtail** | 极轻量、与 Grafana 集成 | 全文检索能力弱 | 已有 Grafana 的团队 |
| **Graylog** | 开箱即用、界面友好 | 扩展性不如 ELK | 中小规模 |
| **Splunk** | 功能强大 | 昂贵（商业软件） | 预算充足的企业 |

---

> **文档版本：** v2.0（ELK 9.4.x）
> **适用环境：** CentOS 7+ / Ubuntu 18.04+
> **最后更新：** 2026-06-03
