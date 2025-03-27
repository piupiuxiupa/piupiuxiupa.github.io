---
title: k8s rbac & Jenkins container
date: 2024-11-22 15:45:09
tags: k8s
categories: Posts
---

# Kubenetes rbac & Jenkins container

## k8s rbac configure

- apiversion: rbac.authorization.k8s.io/v1
- kind
  - ClusterRoleBinding
  - ServiceAccount
  - ClusterRole
- token
  - 容器内：/var/run/secrets/kubernetes.io/serviceaccount/token
- 部署pod spec中添加serviceAccountName: name

## Jenkins configure docker runtime

- 将宿主机的docker.sock挂载到容器内
  - 路径：/var/run/docker.sock
- 确保容器内有docker命令

## Jenkins 自动化部署完整流程

如需操作k8s集群，需要kubectl命令

自动化流程：
- 配置jenkins流水线 - webhook 触发 push event
- git 拉取代码
- docker build 构建镜像
- docker push 推送镜像
- k8s 部署应用
  
```bash
# token 是由rbac配置获取
# Jenkins 中可通过构建环境中的 use secret text or file功能将token进行隐藏
kubectl --server https://xxx --insecure-skip-tls-verify=true --token xxx -n $namespace set image deploy/$deploy_name $container_name=$image_name
```