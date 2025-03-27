---
title: K8s 基础操作与配置
date: 2024-08-27 14:42:39
tags: k8s
categories: Posts
---

# K8s 基础

## kubectl cmd

- command 子命令，常用如 create、 delete、 describe、 get、 apply、 exec等。

- TYPE 资源对象类型，区分大小写。 可简写，简写形式可用`kubectl api-resources`查看。

- NAME 资源对象名称，区分大小写。

- flags 子命令行参数。

```bash
$ kubectl [command] [TYPE] [NAME] [flags]

## 获取多个pod信息
kubectl get pods name1 name2

## 获取多种对象信息
kubectl get pod/pod_name rc/rc_name
kubectl get pod,rc,service

## 同时应用多个YAML文件，以多个-f file参数表示 
kubectl get pod -f pod1.yaml -f pod2.yaml
kubectl create -f pod1.yaml -f rc1.yaml -f service1.yaml

## 按名字进行排序
kubectl get pods --sort-by=.metadata.name

## 查看指定 namespace下的指定标签下的 pod
kubectl get pod -n ns -l <labelname>=nginx_proxy

## 创建资源对象
## 根据 yaml配置文件一次性创建 service和 rc
kubectl create -f my-service.yaml -f my-rc.yaml

## 根据<directory>目录下所有.yaml、.yml、.json文件的定义进行创建
kubectl create -f <directory>

## 删除所有包含某个label的pod和service
kubectl delete pods,services -l name=<label-name>

## 删除所有pod
kubectl delete pods --all

## 指定 Pod中的指定容器执行 date命令
kubectl exec <pod-name> -c <container-name> date

## 登录容器
kubectl exec -it <pod-name> -c <container-name> -- /bin/bash

## 跟踪查看容器日志
kubectl logs -f <pod-name> -c <container-name>

## 创建或更新资源对象
kubectl apply -f app.yaml

## 在线编辑运行中的资源对象
kubectl edit deploy nginx
```
通常情况下不会去使用pod类型单独控制pod，而是使用Deployment、ReplicaSet、StatefulSet等资源对象来配置pod。


## k8s pod config

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: string
  namespace: string
  labels:
    - name: string
  annotations:
    - name: string
spec:
  containers:
  - name: string
    image: string
    imagePullPolicy: [Always | Never | IfNotPresent]
    command: [string]
    args: [string]
    workingDir: string
    volumeMounts:
    - name: string
      mountPath: string
      readOnly: boolean
    ports:
    - name: string
      containerPort: int
      hostPort: int
      protocol: string
    env:
    - name: string
      value: string
    resources:
      limits:
        cpu: string
        memory: string
      requests:
        cpu: string
        memory: string
  volumes:
    - name: string
    emptyDir: {}
    hostPath:
      path: string
    secret:
      secretName: string
      items:
        - key: string
          path: string
    configMap:
      name: string
      items:
      - key: string
        path: string

    livenessProbe:
      exec:
        command: [string]
      httpGet:
        path: string
        host: string
        scheme: string
      httpHeaders:
      - name: string
        value: string
      tcpSocket:
        port: number
      initialDelaySeconds: 0
      timeoutSeconds: 0
      periodSeconds: 0
      successThreshold: 0
      failureThreshold: 0
    securityContext:
      privileged: false
  restartPolicy: [Always | Never | OnFailure]
  nodeSelector: object
  imagePullSecrets:
    - name: string
  hostNetwork: false   
```

## Pod共享volume
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: tomcat
    image: tomcat
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: app-logs
      mountPath: /usr/local/tomcat/logs
  - name: busybox
    image: busybox
    command: ["sh", "-c", "tail -f /logs/catalina*.log"]
    volumeMounts:
    - name: app-logs
      mountPath: /logs
  # 设置的Volume名为app-logs,类型为emptyDir
  volumes:
  - name: app-logs
    emptyDir: {}
```

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-appvars
data:
  apploglevel: info
  appdatadir: /var/data

```

## DaemonSet

用于管理在集群中每个Node上仅运行一份Pod的副本实例
使用场景：
- 在每个node上都运行一个GlusterFS存储或者Ceph存储的Daemon进程
- 在每个Node上都运行一个日志采集程序
- 在每个Node上都运行一个性能监控程序，采集该Node的运行性能数据

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-cloud-logging
  namespace: kube-system
  labels:
    k8s-app: fluentd-cloud-logging
spec:
  template:
    metadata:
      namespace: kube-system
      labels:
      k8s-app: fluentd-cloud-logging
    spec:
      containers:
      - name: fluentd-cloud-logging
      image: gcr.io/google_containers/fluentd-elasticsearch:1.17
      resources:
        limits:
          cpu: 100m
          memory: 200Mi
      env:
      - name: FLUENTD_ARGS
        value: -q
      volumeMounts:
      - name: varlog
        mountPath: /var/log
        readOnly: false
      - name: containers
        mountPath: /var/lib/docker/containers
        readOnly: false
      volumes:
      - name: containers
      hostPath:
        path: /var/lib/docker/containers
      - name: varlog
      hostPath:
        path: /var/log
  updateStrategy:
    type: RollingUpdate
```
> updateStrategy的另外一个值是OnDelete，即只有手工删除了DaemonSet创建的Pod副本，新的Pod副本才会被创建出来

## Job 批处理调度

通过Kubernetes Job资源对象来定义并启动一个批处理任务

批处理可分为：
- Job Template Expansion模式
  - 一个Job对象对应一个待处理的Work item，有几个Work item就产生几个独立的Job
- Queue with Pod Per Work Item模式
  - 采用一个任务队列存放Work item，一个Job对象作为消费者去完成这些Work item
  - Job会启动N个Pod，每个Pod都对应一个Work item
- Queue with Variable Pod Count模式
  - 采用一个任务队列存放Work item，一个Job对象作为消费者去完成这些Work item，但与上面的模式不同，Job启动的Pod数量是可变的。
- Single Job with Static Work Assignment模式
  - 一个Job产生多个Pod，但它采用程序静态方式分配任务项，而不是采用队列模式进行动态分配

Kubernetes将Job分以下三种类型:
- Non-parallel Jobs
- Parallel Jobs with a fixed completion count
- Parallel Jobs with a work queue

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: process-item-$ITEM
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      name: jobexample
      labels:
        jobgroup: jobexample
    spec:
      containers:
      - name: c
        image: busybox
        command: ["sh", "-c", "echo Processing item $ITEM&& sleep 5"]
      restartPolicy: Never
```

## Cronjob 定时任务

基本上可参照Linux Cron的表达式

`Minutes Hours DayofMonth Month DayofWeek Year`

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

## Pod 升级回滚

```bash
# 更新镜像
kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
# 或者edit编辑
kubectl edit deployment/nginx-deployment

# 查看deployment更新过程
kubetl rollout status deployment/hello

## 查看历史记录
kubectl rollout history deployment/hello
# 查看指定版本
kubectl rollout history deployment/hello --revision=2
# 回滚到上一个版本
kubectl rollout undo deployment/hello
# 回滚到指定版本
kubectl rollout undo deployment/hello --to-revision=2

## 暂停deployment更新
kubectl rollout pause deployment/hello
# 恢复Deployment部署
kubectl rollout resume deployment/hello
# 在恢复暂停的Deployment之前，无法回滚该Deployment
```