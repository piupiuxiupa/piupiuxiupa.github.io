---
title: Helm 部署 DolphinScheduler 3.4.1 
date: 2026-04-10 10:00:18
tags: tools
categories: Posts
---

## 目录

- [1. 概述](#1-概述)
- [2. 环境信息](#2-环境信息)
- [3. Helm Chart 结构](#3-helm-chart-结构)
- [4. values.yaml 改动清单](#4-valuesyaml-改动清单)
- [5. 持久化配置](#5-持久化配置)
- [6. 部署命令](#6-部署命令)
- [7. 部署验证](#7-部署验证)
- [8. 问题排查记录](#8-问题排查记录)
- [9. 附录：常用命令](#9-附录常用命令)

---

## 1. 概述

本文档记录 Apache DolphinScheduler 3.4.1 基于 Helm Chart 在 Kubernetes 上的完整部署过程

**核心架构组件：**

| 组件 | 工作负载类型 | 说明 |
|------|-------------|------|
| Master | StatefulSet | 工作流调度引擎，负责 DAG 解析和任务分发 |
| Worker | StatefulSet | 任务执行器，实际运行任务 |
| API | Deployment | RESTful API 服务，前端通过它交互 |
| Alert | Deployment | 告警服务，负责发送各类通知 |
| Schema Initializer | Job (Hook) | 数据库 Schema 初始化/升级，作为 post-install/post-upgrade Hook 运行 |

**依赖组件（通过子 Chart 部署）：**

| 依赖 | 子 Chart | 版本 | 说明 |
|------|----------|------|------|
| MySQL | Bitnami MySQL | 9.4.1 | 元数据库（默认使用 PostgreSQL，本次切换为 MySQL） |
| ZooKeeper | Bitnami ZooKeeper | 11.4.11 | 注册中心，Master/Worker 服务发现 |
| MinIO | Bitnami MinIO | 11.10.13 | S3 兼容对象存储，存储资源文件 |

---

## 2. 环境信息

| 项目 | 值 |
|------|----|
| 平台 | Docker Desktop for Mac |
| Kubernetes | Docker Desktop 内置 K8s |
| Helm 版本 | v3.x |
| 命名空间 | `dolphinscheduler` |
| StorageClass | `hostpath` (default), provisioner: `docker.io/hostpath` |
| PVC 数据位置 | Docker Desktop Linux VM `/var/lib/k8s-pvs/` |
| 数据库 | MySQL 8.0（通过 Bitnami 子 Chart 部署） |
| 注册中心 | ZooKeeper 3.8.4（通过 Bitnami 子 Chart 部署） |
| 对象存储 | MinIO（通过 Bitnami 子 Chart 部署，演示用途） |
| 镜像仓库 | `apache` (Docker Hub) |
| DolphinScheduler 版本 | 3.4.1 |

---

## 3. Helm Chart 结构

```
dolphinscheduler-helm/
├── Chart.yaml                          # Chart 元数据，appVersion: 3.4.1
├── Chart.lock                          # 依赖锁定文件
├── values.yaml                         # 主配置文件（已修改）
├── README.md                           # 自动生成的参数说明
├── README.md.gotmpl                    # README 模板
├── plugins-init/                       # 自定义插件初始化镜像构建文件
│   ├── Dockerfile                      # 构建 dolphinscheduler-plugins-init 镜像
│   ├── Dockerfile.curl                 # 备选 Dockerfile（基于 curl 下载）
│   ├── connector/                      # 数据库连接器 JAR 包
│   │   └── mysql-connector-java-*.jar
│   └── plugins/                        # 自定义任务插件
├── charts/                             # 依赖子 Chart
│   ├── mysql/                          # Bitnami MySQL 9.4.1
│   ├── zookeeper/                      # Bitnami ZooKeeper 11.4.11
│   ├── minio/                          # Bitnami MinIO 11.10.13
│   └── postgresql/                     # Bitnami PostgreSQL 10.3.18（未启用）
└── templates/                          # Kubernetes 资源模板
    ├── _helpers.tpl                    # 模板辅助函数
    ├── configmap.yaml                  # 通用 ConfigMap
    ├── configmap-dolphinscheduler-common.yaml   # 通用环境变量 ConfigMap
    ├── configmap-dolphinscheduler-master.yaml
    ├── configmap-dolphinscheduler-worker.yaml
    ├── configmap-dolphinscheduler-alert.yaml
    ├── configmap-dolphinscheduler-api.yaml
    ├── statefulset-dolphinscheduler-master.yaml
    ├── statefulset-dolphinscheduler-worker.yaml
    ├── deployment-dolphinscheduler-alert.yaml
    ├── deployment-dolphinscheduler-api.yaml
    ├── job-dolphinscheduler-schema-initializer.yaml  # DB Schema 初始化 Job
    ├── pvc-dolphinscheduler-shared.yaml
    ├── pvc-dolphinscheduler-fs-file.yaml
    ├── pvc-dolphinscheduler-alert.yaml
    ├── pvc-dolphinscheduler-api.yaml
    ├── svc-dolphinscheduler-api.yaml
    ├── svc-dolphinscheduler-alert.yaml
    ├── svc-dolphinscheduler-master-headless.yaml
    ├── svc-dolphinscheduler-worker-headless.yaml
    ├── secret-external-database.yaml
    ├── secret-registry-database.yaml
    ├── secret-external-etcd-ssl.yaml
    ├── secret-external-ldap-ssl.yaml
    ├── rbac.yaml
    ├── ingress.yaml
    ├── keda-autoscaler-worker.yaml
    └── NOTES.txt                        # 安装后提示信息
```

### Schema Initializer 工作机制

`job-dolphinscheduler-schema-initializer.yaml` 定义了一个 Helm Hook Job：

- **触发时机**: `post-install`, `post-upgrade`, `post-rollback`
- **执行脚本**: `tools/bin/upgrade-schema.sh`
- **升级方式**: 增量式 Schema 迁移，使用 `CREATE IF NOT EXISTS` / `ALTER` 语句，不会覆盖已有数据
- **MySQL 支持**: 当使用 MySQL 时，会自动通过 `dolphinscheduler-plugins-init` 镜像复制 MySQL Connector JAR 包到 tools 的 libs 目录

---

## 4. values.yaml 改动清单

以下是与上游默认值相比的所有修改项，按类别分组。

### 4.1 镜像与数据源

| 配置项 | 默认值 | 当前值 | 说明 |
|--------|--------|--------|------|
| `image.tag` | `"latest"` | `"3.4.1"` | 锁定版本号 |
| `datasource.profile` | `"postgresql"` | `"mysql"` | 切换到 MySQL 数据源 |

### 4.2 数据库配置

| 配置项 | 默认值 | 当前值 | 说明 |
|--------|--------|--------|------|
| `postgresql.enabled` | `true` | `false` | 关闭内置 PostgreSQL |
| `postgresql.persistence.enabled` | `false` | `true` | 启用 PostgreSQL 持久化（虽已禁用 PostgreSQL，保留配置以防后续切换） |
| `postgresql.persistence.storageClass` | `"-"` | `"hostpath"` | StorageClass 名称 |
| `mysql.enabled` | `false` | `true` | 启用内置 MySQL |
| `mysql.auth.username` | - | `"ds"` | MySQL 用户名 |
| `mysql.auth.password` | - | `"ds"` | MySQL 密码 |
| `mysql.primary.persistence.enabled` | `false` | `true` | 启用 MySQL 数据持久化 |
| `mysql.primary.persistence.storageClass` | `"-"` | `"hostpath"` | StorageClass 名称 |

### 4.3 存储配置

| 配置项 | 默认值 | 当前值 | 说明 |
|--------|--------|--------|------|
| `minio.enabled` | `true` | `true` | 保持不变 |
| `minio.persistence.enabled` | `false` | `false` | 保持不变，演示环境不持久化 MinIO |
| `zookeeper.enabled` | `true` | `true` | 保持不变 |
| `zookeeper.persistence.enabled` | `false` | `true` | 启用 ZK 数据持久化 |
| `zookeeper.persistence.storageClass` | `"-"` | `"hostpath"` | StorageClass 名称 |

### 4.4 公共存储

| 配置项 | 默认值 | 当前值 | 说明 |
|--------|--------|--------|------|
| `common.sharedStoragePersistence.storageClassName` | `"-"` | `"hostpath"` | 共享存储（Hadoop/Spark 等二进制包） |
| `common.fsFileResourcePersistence.storageClassName` | `"-"` | `"hostpath"` | 文件资源存储 |

### 4.5 Master 组件

| 配置项 | 默认值 | 当前值 | 说明 |
|--------|--------|--------|------|
| `master.replicas` | `"3"` | `"1"` | 开发环境缩减为 1 副本 |
| `master.persistentVolumeClaim.enabled` | `false` | `true` | 启用 Master PVC |
| `master.persistentVolumeClaim.storageClassName` | `"-"` | `"hostpath"` | StorageClass 名称 |
| `master.initContainers` | `{}` | *(见下方)* | 添加插件初始化容器 |
| `master.extraVolumeMounts` | - | *(见下方)* | 挂载插件卷 |
| `master.extraVolumes` | - | *(见下方)* | 定义插件卷 |

### 4.6 Worker 组件

| 配置项 | 默认值 | 当前值 | 说明 |
|--------|--------|--------|------|
| `worker.replicas` | `"3"` | `"1"` | 开发环境缩减为 1 副本 |
| `worker.persistentVolumeClaim.enabled` | `false` | `true` | 启用 Worker PVC 总开关 |
| `worker.persistentVolumeClaim.dataPersistentVolume.enabled` | `false` | `true` | 启用数据卷 |
| `worker.persistentVolumeClaim.dataPersistentVolume.storageClassName` | `"-"` | `"hostpath"` | 数据卷 StorageClass |
| `worker.persistentVolumeClaim.logsPersistentVolume.enabled` | `false` | `true` | 启用日志卷 |
| `worker.persistentVolumeClaim.logsPersistentVolume.storageClassName` | `"-"` | `"hostpath"` | 日志卷 StorageClass |
| `worker.initContainers` | `{}` | *(见下方)* | 添加插件初始化容器 |
| `worker.extraVolumeMounts` | - | *(见下方)* | 挂载插件卷 |
| `worker.extraVolumes` | - | *(见下方)* | 定义插件卷 |

### 4.7 Alert 组件

| 配置项 | 默认值 | 当前值 | 说明 |
|--------|--------|--------|------|
| `alert.persistentVolumeClaim.enabled` | `false` | `true` | 启用 Alert PVC |
| `alert.persistentVolumeClaim.storageClassName` | `"-"` | `"hostpath"` | StorageClass 名称 |
| `alert.initContainers` | - | *(见下方)* | 添加插件初始化容器 |
| `alert.extraVolumeMounts` | - | *(见下方)* | 挂载插件卷 |
| `alert.extraVolumes` | - | *(见下方)* | 定义插件卷 |

### 4.8 API 组件

| 配置项 | 默认值 | 当前值 | 说明 |
|--------|--------|--------|------|
| `api.persistentVolumeClaim.enabled` | `false` | `true` | 启用 API PVC |
| `api.persistentVolumeClaim.storageClassName` | `"-"` | `"hostpath"` | StorageClass 名称 |
| `api.initContainers` | - | *(见下方)* | 添加插件初始化容器 |
| `api.extraVolumeMounts` | - | *(见下方)* | 挂载插件卷 |
| `api.extraVolumes` | - | *(见下方)* | 定义插件卷 |

### 4.9 Init Containers 配置详情

四个组件（Master/Worker/Alert/API）的 initContainers 配置模式相同，区别仅在于基础镜像和 libs 路径：

**Master 示例：**

```yaml
master:
  initContainers:
    - name: copy-libs
      image: apache/dolphinscheduler-master:3.4.1
      imagePullPolicy: IfNotPresent
      command:
        - /bin/sh
        - -c
        - |
          echo "Copying original libs..."
          cp -r /opt/dolphinscheduler/master-server/libs/* /target-libs/
      volumeMounts:
        - name: plugins-libs
          mountPath: /target-libs
    - name: plugins-init
      image: apache/dolphinscheduler-plugins-init:3.4.1
      imagePullPolicy: IfNotPresent
      command:
        - /bin/sh
        - -c
        - |
          echo "Copying connector jars..."
          cp /connector/*.jar /target-libs/
          echo "Copying plugins..."
          cp -r /plugins/* /target-plugins/
      volumeMounts:
        - name: plugins-libs
          mountPath: /target-libs
        - name: plugins-plugins
          mountPath: /target-plugins
  extraVolumeMounts:
    - name: plugins-libs
      mountPath: /opt/dolphinscheduler/master-server/libs
    - name: plugins-plugins
      mountPath: /opt/dolphinscheduler/plugins
  extraVolumes:
    - name: plugins-libs
      emptyDir: {}
    - name: plugins-plugins
      emptyDir: {}
```

**各组件的 libs 路径差异：**

| 组件 | 基础镜像 | libs 路径 |
|------|----------|-----------|
| Master | `dolphinscheduler-master:3.4.1` | `/opt/dolphinscheduler/master-server/libs` |
| Worker | `dolphinscheduler-worker:3.4.1` | `/opt/dolphinscheduler/worker-server/libs` |
| Alert | `dolphinscheduler-alert-server:3.4.1` | `/opt/dolphinscheduler/alert-server/libs` |
| API | `dolphinscheduler-api:3.4.1` | `/opt/dolphinscheduler/api-server/libs` |

**工作原理：**

1. `copy-libs` 容器先将原始镜像的 libs 复制到 `plugins-libs` emptyDir 卷
2. `plugins-init` 容器再将 MySQL Connector JAR 和自定义插件追加到同一个卷
3. 主容器启动时挂载该卷，覆盖原始 libs 目录，同时获得原始依赖 + MySQL 驱动 + 自定义插件

---

## 5. 持久化配置

### 5.1 StorageClass 命名差异

本 Chart 混合使用了两种不同的子 Chart 来源，它们的 StorageClass 配置键名不一致，这是一个容易踩坑的地方：

| 来源 | 组件 | 配置键名 | 示例 |
|------|------|----------|------|
| Bitnami 子 Chart | MySQL, ZooKeeper, PostgreSQL, MinIO | `storageClass` | `mysql.primary.persistence.storageClass` |
| DolphinScheduler 原生 | Master, Worker, Alert, API | `storageClassName` | `master.persistentVolumeClaim.storageClassName` |

**Bitnami 子 Chart** 在模板内部会将 `storageClass` 映射为 K8s PVC 的 `storageClassName` 字段。所以虽然配置键名不同，最终效果一致。

### 5.2 默认值 `"-"` 的含义

默认配置中，所有 `storageClass` / `storageClassName` 的值都是 `"-"`。这意味着设置 `storageClassName: ""`，即**禁用动态供给**。如果要让 K8s 自动创建 PV，必须指定一个实际存在的 StorageClass 名称。

### 5.3 PVC 启用条件

要让某个组件的 PVC 真正创建出来，需要**同时满足**两个条件：

1. `enabled: true` （开启 PVC）
2. `storageClass` / `storageClassName` 设置为集群中实际存在的 StorageClass 名称

只改了 `storageClass` 但忘记设 `enabled: true`，PVC 不会创建。这是一个常见的配置失误。

### 5.4 已创建的 PVC

部署后在 `dolphinscheduler` 命名空间中实际创建的 PVC：

| PVC 名称 | 组件 | 状态 | VM 数据路径 |
|-----------|------|------|-------------|
| `dolphinscheduler-api` | API Server | Bound | `/var/lib/k8s-pvs/dolphinscheduler-api/` |
| `dolphinscheduler-alert` | Alert Server | Bound | `/var/lib/k8s-pvs/dolphinscheduler-alert/` |
| `data-dolphinscheduler-mysql-0` | MySQL | Bound | `/var/lib/k8s-pvs/data-dolphinscheduler-mysql-0/` |
| `dolphinscheduler-master-dolphinscheduler-master-0` | Master | Bound | `/var/lib/k8s-pvs/dolphinscheduler-master-dolphinscheduler-master-0/` |
| `data-dolphinscheduler-zookeeper-0` | ZooKeeper | Bound | `/var/lib/k8s-pvs/data-dolphinscheduler-zookeeper-0/` |

> 注意：MinIO 的 `persistence.enabled` 仍为 `false`，所以没有 MinIO PVC。Worker 的 PVC 状态取决于最终配置。

### 5.5 PVC 访问模式

| PVC 类型 | 访问模式 | 说明 |
|----------|----------|------|
| Master, Worker, Alert, API | `ReadWriteOnce` | 单节点读写，StatefulSet/Deployment 适用 |
| Worker Data, Worker Logs | `ReadWriteOnce` | 单节点读写 |
| 共享存储 (sharedStorage) | `ReadWriteMany` | 多节点共享，需支持 RWM 的 StorageClass |
| 文件资源 (fsFileResource) | `ReadWriteMany` | 多节点共享 |

---

## 6. 部署命令

### 6.1 前置准备

```bash
# 构建插件初始化镜像
cd plugins-init && docker buildx build --platform linux/amd64 -t apache/dolphinscheduler-plugins-init:3.4.1 .

# 确认 K8s 集群可用
kubectl cluster-info

# 确认 StorageClass
kubectl get sc
# 预期输出：hostpath (default)

# 创建命名空间
kubectl create namespace dolphinscheduler
```

### 6.2 首次部署（Helm Install）

```bash
# 从 Chart 根目录执行
helm install dolphinscheduler . \
  --namespace dolphinscheduler \
  -f values.yaml
```

如果需要通过命令行覆盖个别参数（不修改 values.yaml）：

```bash
helm install dolphinscheduler . \
  --namespace dolphinscheduler \
  --set image.tag=3.4.1 \
  --set datasource.profile=mysql \
  --set mysql.enabled=true \
  --set postgresql.enabled=false
```

### 6.3 升级部署（Helm Upgrade）

```bash
# 修改 values.yaml 后执行
helm upgrade dolphinscheduler . \
  --namespace dolphinscheduler \
  -f values.yaml
```

### 6.4 卸载

```bash
helm uninstall dolphinscheduler --namespace dolphinscheduler

# 如需清理 PVC（会删除所有数据）
kubectl delete pvc --all --namespace dolphinscheduler
```

### 6.5 端口转发访问

由于 API Server 默认使用 ClusterIP，本地访问需要端口转发：

```bash
kubectl port-forward svc/dolphinscheduler-api 12345:12345 \
  --namespace dolphinscheduler
```

然后浏览器访问 `http://localhost:12345/dolphinscheduler`

默认登录账号：`admin / dolphinscheduler123`

---

## 7. 部署验证

### 7.1 检查所有 Pod 状态

```bash
kubectl get pods -n dolphinscheduler -o wide
```

预期所有 Pod 都为 `Running` 状态，READY 列显示 `1/1`。

### 7.2 检查 Schema 初始化 Job

```bash
# 查看 Job 执行状态
kubectl get jobs -n dolphinscheduler

# 查看 Job 日志
kubectl logs job/dolphinscheduler-db-init-job -n dolphinscheduler
```

Schema 初始化 Job 应显示 `Completed` 状态。如果 Job 失败，各组件的 Pod 可能无法正常启动。

### 7.3 检查 PVC 绑定

```bash
kubectl get pvc -n dolphinscheduler
```

所有 PVC 应为 `Bound` 状态。

### 7.4 检查服务端点

```bash
kubectl get svc -n dolphinscheduler
```

确认 `dolphinscheduler-api` 服务存在。

### 7.5 验证 MySQL 连接

```bash
# 进入 MySQL Pod
kubectl exec -it dolphinscheduler-mysql-0 -n dolphinscheduler -- bash

# 在 MySQL 内部
mysql -uds -pds dolphinscheduler

# 检查 Schema 版本
SELECT * FROM t_ds_version;
```

### 7.6 检查各组件日志

```bash
# Master 日志
kubectl logs statefulset/dolphinscheduler-master -n dolphinscheduler --tail=100

# Worker 日志
kubectl logs statefulset/dolphinscheduler-worker -n dolphinscheduler --tail=100

# API 日志
kubectl logs deployment/dolphinscheduler-api -n dolphinscheduler --tail=100

# Alert 日志
kubectl logs deployment/dolphinscheduler-alert -n dolphinscheduler --tail=100
```

---

## 8. 问题排查记录

### 8.1 PVC 未创建

**现象**：修改了 `storageClass` 为 `hostpath`，但部署后没有看到对应的 PVC。

**原因**：`enabled` 字段仍然是 `false`。

**解决**：必须同时设置 `enabled: true` 和正确的 StorageClass 名称。

```yaml
# 错误：只改了 storageClass
master:
  persistentVolumeClaim:
    enabled: false          # <-- 漏掉了这个
    storageClassName: "hostpath"

# 正确：两个都要改
master:
  persistentVolumeClaim:
    enabled: true
    storageClassName: "hostpath"
```

### 8.2 Docker Desktop VM 数据位置

**现象**：需要查看 PVC 的实际存储数据，但在 macOS 文件系统中找不到。

**原因**：Docker Desktop 的 K8s 运行在一个 Linux 虚拟机中，PVC 数据存储在 VM 内部，macOS 文件系统不可见。

**进入 Docker Desktop VM**：

```bash
docker run -it --rm --privileged --pid=host alpine nsenter -t 1 -m -u -n -i sh
```

进入 VM 后查看 PVC 数据：

```bash
ls /var/lib/k8s-pvs/
# dolphinscheduler-api/
# dolphinscheduler-alert/
# data-dolphinscheduler-mysql-0/
# dolphinscheduler-master-dolphinscheduler-master-0/
# data-dolphinscheduler-zookeeper-0/
```

**macOS 与 VM 的共享目录**：

Docker Desktop 会将以下 macOS 目录挂载到 VM 中：
- `/Users`
- `/Volumes`
- `/tmp`
- `/private`

这意味着你可以把 PVC 数据复制到这些共享目录来从 macOS 访问。

---

## 9. 附录：常用命令

### 集群信息

```bash
# 查看集群信息
kubectl cluster-info

# 查看节点
kubectl get nodes -o wide

# 查看 StorageClass
kubectl get sc

# 查看 StorageClass 详情
kubectl describe sc hostpath
```

### 命名空间操作

```bash
# 创建命名空间
kubectl create namespace dolphinscheduler

# 查看命名空间
kubectl get namespaces

# 设置默认命名空间（方便操作）
kubectl config set-context --current --namespace=dolphinscheduler
```

### Pod 管理

```bash
# 查看所有 Pod
kubectl get pods -n dolphinscheduler -o wide

# 查看 Pod 事件
kubectl describe pod <pod-name> -n dolphinscheduler

# 查看 Pod 日志
kubectl logs <pod-name> -n dolphinscheduler --tail=200

# 实时跟踪日志
kubectl logs <pod-name> -n dolphinscheduler -f

# 进入 Pod
kubectl exec -it <pod-name> -n dolphinscheduler -- bash
```

### PVC 管理

```bash
# 查看所有 PVC
kubectl get pvc -n dolphinscheduler

# 查看 PVC 详情
kubectl describe pvc <pvc-name> -n dolphinscheduler

# 查看 PV
kubectl get pv

# 删除所有 PVC（危险操作）
kubectl delete pvc --all -n dolphinscheduler
```

### Helm 操作

```bash
# 查看已安装的 Release
helm list -n dolphinscheduler

# 查看 Release 状态
helm status dolphinscheduler -n dolphinscheduler

# 查看 Release 历史
helm history dolphinscheduler -n dolphinscheduler

# 回滚到上一个版本
helm rollback dolphinscheduler -n dolphinscheduler

# 回滚到指定版本
helm rollback dolphinscheduler <revision> -n dolphinscheduler

# 查看 Release 的 values
helm get values dolphinscheduler -n dolphinscheduler

# 渲染模板（不安装，用于调试）
helm template dolphinscheduler . --namespace dolphinscheduler -f values.yaml > rendered.yaml
```

### MySQL 操作

```bash
# 进入 MySQL Pod
kubectl exec -it dolphinscheduler-mysql-0 -n dolphinscheduler -- bash

# 连接 MySQL
mysql -uds -pds dolphinscheduler

# 查看版本
SELECT version();

# 查看 DolphinScheduler Schema 版本
SELECT * FROM t_ds_version;

# 查看所有表
SHOW TABLES;

# 查看表结构
DESC t_ds_user;

# 查看表行数
SELECT table_name, table_rows
FROM information_schema.tables
WHERE table_schema = 'dolphinscheduler'
ORDER BY table_rows DESC;
```

### Docker Desktop VM 操作

```bash
# 进入 VM
docker run -it --rm --privileged --pid=host alpine nsenter -t 1 -m -u -n -i sh

# 在 VM 中查看 PVC 数据
ls -la /var/lib/k8s-pvs/

# 在 VM 中查看磁盘使用
du -sh /var/lib/k8s-pvs/*

# 从 VM 复制数据到 macOS 共享目录
cp -r /var/lib/k8s-pvs/dolphinscheduler-mysql-0/ /Users/yourname/Desktop/mysql-backup/
```

### 资源清理

```bash
# 完全卸载（包括 PVC）
helm uninstall dolphinscheduler -n dolphinscheduler
kubectl delete pvc --all -n dolphinscheduler

# 删除命名空间
kubectl delete namespace dolphinscheduler

# 如果有残留资源
kubectl get all -n dolphinscheduler
kubectl delete all --all -n dolphinscheduler
```

---

## 附：使用外部 MySQL 和 ZooKeeper 的配置

如果不想使用内置的 MySQL 和 ZooKeeper，而是连接已有的外部服务，配置如下：

```yaml
# 关闭内置数据库和注册中心
postgresql:
  enabled: false
mysql:
  enabled: false
zookeeper:
  enabled: false

# 配置外部数据库
externalDatabase:
  enabled: true
  type: "mysql"
  host: "your-mysql-host"
  port: "3306"
  username: "your-username"
  password: "your-password"
  database: "dolphinscheduler"
  params: "characterEncoding=utf8"
  driverClassName: "com.mysql.cj.jdbc.Driver"

# 配置外部注册中心
externalRegistry:
  registryPluginName: "zookeeper"
  registryServers: "your-zk-host:2181"
```

使用外部数据库时，Schema Initializer Job 仍然会作为 post-install Hook 运行，执行 `upgrade-schema.sh` 进行增量 Schema 迁移。它不会覆盖已有数据。
