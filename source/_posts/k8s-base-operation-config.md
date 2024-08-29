---
title: K8s 基础操作与配置
date: 2024-08-27 14:42:39
tags: k8s
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
