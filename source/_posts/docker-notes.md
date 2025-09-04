---
title: Docker Notes
date: 2024-11-29 09:25:12
tags: docker
categories: Notes
---

# Docker Notes

## docker save 和 export 区别

1. 功能和用途

   - docker export

     - 导出一个容器的文件系统。
     - 提取容器当前的文件系统快照，不包括镜像的历史层、元数据或启动命令等信息。
     - 通常用于备份或迁移容器的文件系统内容。

   - docker save
     - 保存一个镜像及其历史层和元数据。
     - 包括镜像的所有历史层、CMD、ENTRYPOINT、环境变量等配置。
     - 通常用于备份或迁移镜像（可以在目标环境中直接加载并运行）。

2. 导出的内容

   - docker export
     - 导出的内容是容器的文件系统，形式是一个纯文件层，没有镜像的结构信息。
     - 不包括启动命令、环境变量等元数据。
   - docker save
     - 导出的内容是镜像的完整结构，包括所有层和元数据。
     - 可直接用于加载到其他 Docker 实例中。

3. 使用场景

   - docker export
     - 当您只需要容器文件系统中的文件，而不需要镜像信息时使用。
     - 用于容器数据备份或在不需要 Docker 环境的情况下提取文件。
   - docker save
     - 用于迁移完整镜像，包括历史记录、配置和多层结构。
     - 常用于跨环境传输镜像，例如将镜像从开发环境迁移到生产环境。

4. 加载/恢复方式

   - docker export

     - 使用 tar 工具解压提取内容，或者通过 docker import 恢复为一个新镜像，但只包含文件系统，没有元数据。

     ```bash
     docker import container.tar <new_image_name>
     ```

   - docker save

     - 使用 docker load 加载为完整镜像，可以直接运行。

     ```bash
     docker load -i image.tar
     ```

## 容器共享内存

在 Docker 容器中，/dev/shm 是一个内存共享区域，通常用于高效的进程间通信（IPC）。Docker 默认会为每个容器分配一个大小固定的 /dev/shm 区域，默认大小为 64MB，这在某些应用场景（如高性能计算、数据库操作等）中可能不足。

1. /dev/shm 的默认大小：

   默认大小为 64MB，但可以通过配置容器启动参数增加。

2. 使用场景：

   - 数据库（如 PostgreSQL、Redis）：需要使用共享内存进行缓存。

   - 图像处理（如 OpenCV）：某些库需要依赖共享内存。

   - 浏览器自动化（如 Selenium）：浏览器运行依赖共享内存。

3. 查看容器内 /dev/shm 大小：df -h /dev/shm

### 调整共享内存

1. 在 docker run 中指定

   ```bash
   docker run --shm-size=1g your_image
   ```

2. 在 docker-compose 中配置

   ```yaml
   services:
     your_service:
       image: your_image
       shm_size: 1g
   ```

3. 在 Kubernetes 中配置

   在 Pod 的 spec 中，使用 emptyDir 配置共享内存。例如：

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: example-deployment
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: example-app
     template:
       metadata:
         labels:
           app: example-app
       spec:
         containers:
           - name: example-container
             image: your-image:latest
             resources:
               limits:
                 memory: 2Gi
             volumeMounts:
               - mountPath: /dev/shm
                 name: dshm
         volumes:
           - name: dshm
             emptyDir:
               medium: Memory
               sizeLimit: 1Gi
   ```

### 检查共享内存使用情况

```bash
ipcs -m
```

### 指定平台获取镜像及构建

```bash
# 设置环境变量让docker默认获取x86_64架构的镜像
export DOCKER_DEFAULT_PLATFORM=linux/amd64

# 指定构建的指令平台
docker buildx build --platform linux/amd64 -t your-image:latest .

# 拉取x86_64架构的ubuntu镜像
docker pull --platform=linux/amd64 ubuntu:22.04
```
