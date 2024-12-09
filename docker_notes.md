# Docker Notes

## 容器共享内存

在 Docker 容器中，/dev/shm 是一个内存共享区域，通常用于高效的进程间通信（IPC）。Docker 默认会为每个容器分配一个大小固定的 /dev/shm 区域，默认大小为 64MB，这在某些应用场景（如高性能计算、数据库操作等）中可能不足。

	1.	/dev/shm 的默认大小：
默认大小为 64MB，但可以通过配置容器启动参数增加。
	2.	使用场景：
	•	数据库（如 PostgreSQL、Redis）：需要使用共享内存进行缓存。
	•	图像处理（如 OpenCV）：某些库需要依赖共享内存。
	•	浏览器自动化（如 Selenium）：浏览器运行依赖共享内存。
	3.	查看容器内 /dev/shm 大小：df -h /dev/shm


1. 在 docker run 中指定
docker run --shm-size=1g your_image
2. 在 docker-compose 中配置
services:
  your_service:
    image: your_image
    shm_size: 1g

3. 在 Kubernetes 中配置（如果使用容器编排）
在 Pod 的 spec 中，使用 emptyDir 配置共享内存。例如：

apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: example-container
    image: your_image
    volumeMounts:
    - mountPath: /dev/shm
      name: dshm
  volumes:
  - name: dshm
    emptyDir:
      medium: Memory
      sizeLimit: 1Gi


检查共享内存使用情况
ipcs -m