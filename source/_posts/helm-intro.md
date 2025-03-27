---
title: Helm 初探
date: 2024-12-20 16:32:38
tags: tools
categories: Posts
---

# helm 初探

> https://helm.sh/zh/docs/intro/using_helm/

## 三大概念

Chart 代表着 Helm 包。它包含在 Kubernetes 集群内部运行应用程序，工具或服务所需的所有资源定义。你可以把它看作是 Homebrew formula，Apt dpkg，或 Yum RPM 在Kubernetes 中的等价物。

Repository（仓库） 是用来存放和共享 charts 的地方。它就像 Perl 的 CPAN 档案库网络 或是 Fedora 的 软件包仓库，只不过它是供 Kubernetes 包所使用的。

Release 是运行在 Kubernetes 集群中的 chart 的实例。一个 chart 通常可以在同一个集群中安装多次。每一次安装都会创建一个新的 release。以 MySQL chart为例，如果你想在你的集群中运行两个数据库，你可以安装该chart两次。每一个数据库都会拥有它自己的 release 和 release name。

## 初始化

```bash
# 查看当前配置的仓库地址
helm repo list
# 删除默认仓库，默认在国外pull很慢
helm repo remove stable
# 添加几个常用的仓库,可自定义名字
helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
helm repo add kaiyuanshe http://mirror.kaiyuanshe.cn/kubernetes/charts
helm repo add azure http://mirror.azure.cn/kubernetes/charts
helm repo add dandydev https://dandydeveloper.github.io/charts
helm repo add bitnami https://charts.bitnami.com/bitnami
```

## 搜索 chart

Helm 可以从两种来源中进行搜索：

`helm search hub` 从 Artifact Hub 中查找并列出 helm charts。 Artifact Hub中存放了大量不同的仓库。

`helm search repo` 从你添加（使用 helm repo add）到本地 helm 客户端中的仓库中进行查找。该命令基于本地数据进行搜索，无需连接互联网。

```bash
helm search hub wordpress

helm repo update 
helm search repo redis
# 拉取chart包到本地
helm pull bitnami/redis-cluster --version 8.1.2
```

## 安装前修改配置

```bash
helm show values bitnami/wordpress
```

## 安装 helm 包    

```bash
# 安装redis-ha集群，取名redis-ha，需要指定持存储类
helm install redis-cluster bitnami/redis-cluster --set global.storageClass=nfs,global.redis.password=xiagao --version 8.1.2

# 追踪 release 的状态，或是重新读取配置信息
helm status redis-cluster

# 卸载
helm uninstall redis-cluster
```

