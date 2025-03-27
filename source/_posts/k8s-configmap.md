---
title: K8s ConfigMap
date: 2024-12-23 10:44:10
tags: k8s
categories: Posts
---

# ConfigMap

## 基于一个目录来创建 ConfigMap

基于目录来创建 ConfigMap 时，kubectl 识别目录下文件名可以作为合法键名的文件， 并将这些文件打包到新的 ConfigMap 中。普通文件之外的所有目录项都会被忽略 （例如：子目录、符号链接、设备、管道等等）。


```bash
mkdir -p configure-pod-container/configmap/

# 将示例文件下载到 `configure-pod-container/configmap/` 目录
wget https://kubernetes.io/examples/configmap/game.properties -O configure-pod-container/configmap/game.properties
wget https://kubernetes.io/examples/configmap/ui.properties -O configure-pod-container/configmap/ui.properties

# 创建 ConfigMap
kubectl create configmap game-config --from-file=configure-pod-container/configmap/

# 查看 ConfigMap 详情
kubectl describe configmaps game-config
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2022-02-18T18:52:05Z
  name: game-config
  namespace: default
  resourceVersion: "516"
  uid: b4952dc3-d670-11e5-8cd0-68f728db1985
data:
  game.properties: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30    
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice    
```

## 基于文件创建 ConfigMap

```bash
kubectl create configmap game-config-2 --from-file=configure-pod-container/configmap/game.properties

kubectl create configmap game-config-2 --from-file=configure-pod-container/configmap/game.properties --from-file=configure-pod-container/configmap/ui.properties
```

**使用 --from-env-file 选项基于 env 文件创建 ConfigMap**

每行都是一个环境变量，格式为 VAR=VAL。

```bash
# Env 文件包含环境变量列表。其中适用以下语法规则:
# 这些语法规则适用：
#   Env 文件中的每一行必须为 VAR=VAL 格式。
#   以＃开头的行（即注释）将被忽略。
#   空行将被忽略。
#   引号不会被特殊处理（即它们将成为 ConfigMap 值的一部分）。

# 将示例文件下载到 `configure-pod-container/configmap/` 目录
wget https://kubernetes.io/examples/configmap/game-env-file.properties -O configure-pod-container/configmap/game-env-file.properties
wget https://kubernetes.io/examples/configmap/ui-env-file.properties -O configure-pod-container/configmap/ui-env-file.properties

# Env 文件 `game-env-file.properties` 如下所示
cat configure-pod-container/configmap/game-env-file.properties
enemies=aliens
lives=3
allowed="true"
# 此注释和上方的空行将被忽略

kubectl create configmap game-config-env-file \
       --from-env-file=configure-pod-container/configmap/game-env-file.properties
```

## 使用 ConfigMap 数据定义容器环境变量

### 使用单个 ConfigMap 中的数据定义容器环境变量

```bash
kubectl create configmap special-config --from-literal=special.how=very
```

将 ConfigMap 中定义的 special.how 赋值给 Pod 规约中的 SPECIAL_LEVEL_KEY 环境变量。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        # 定义环境变量
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              # ConfigMap 包含你要赋给 SPECIAL_LEVEL_KEY 的值
              name: special-config
              # 指定与取值相关的键名
              key: special.how
  restartPolicy: Never
```

### 使用来自多个 ConfigMap 的数据定义容器环境变量

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: env-config
              key: log_level
  restartPolicy: Never
```

### 将 ConfigMap 中的所有键值对配置为容器环境变量 

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - configMapRef:
          name: special-config
  restartPolicy: Never
```

### 将 ConfigMap 数据添加到一个卷中

如基于文件创建 ConfigMap 中所述，当你使用 --from-file 创建 ConfigMap 时，文件名成为存储在 ConfigMap 的 data 部分中的键， 文件内容成为键对应的值。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        # 提供包含要添加到容器中的文件的 ConfigMap 的名称
        name: special-config
  restartPolicy: Never
```

Pod 运行时，命令 ls /etc/config/ 产生下面的输出:

```
SPECIAL_LEVEL
SPECIAL_TYPE
```

> 如果该容器镜像的 /etc/config 目录中有一些文件，卷挂载将覆盖这些文件。

### 将 ConfigMap 数据添加到卷中的特定路径

使用 path 字段为特定的 ConfigMap 项目指定预期的文件路径。 在这里，ConfigMap 中键 SPECIAL_LEVEL 的内容将挂载在 config-volume 卷中 /etc/config/keys 文件中。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox
      command: [ "/bin/sh","-c","cat /etc/config/keys" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: SPECIAL_LEVEL
          path: keys
  restartPolicy: Never
```

## 使用 subPath

volumeMounts[*].subPath 属性可用于指定所引用的卷内的子路径，而不是其根路径。

使用 subPath 字段将 ConfigMap 中的特定项目挂载到容器中的特定路径。 在这里，ConfigMap 中键 game.properties 的内容将挂载在 config-volume 卷中 /usr/game.properties 文件中。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: subpath-test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      imagePullPolicy: IfNotPresent
      command: [ "/bin/sh","-c","cd /usr && ls && cat /usr/keys" ]
      volumeMounts:
      - name: config-volume
        mountPath: /usr/game.properties
        subPath: game.properties
      - name: config-volume
        mountPath: /usr/ui.properties
        subPath: ui.properties
  volumes:
    - name: config-volume
      configMap:
        name: game-config
  restartPolicy: Never
```

> 以subPath方式挂载时，configmap更新，容器不会更新