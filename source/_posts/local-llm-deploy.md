---
title: 本地大模型部署
date: 2024-08-04 18:59:10
tags: tools
categories: Posts
---

# 本地大模型部署

> Windows根据官网下载安装即可
>
> [Linux 官方安装步骤](https://github.com/ollama/ollama/blob/main/docs/linux.md)

## 1. 安装ollama

```bash
## 可能会比较慢，需要magic
curl -fsSL https://ollama.com/install.sh | sh
## 也可先把ollama-linux-amd64下载到本地
## 然后上传到linux
## 修改安装脚本两处地方再安装
curl -fsSL https://ollama.com/install.sh >> ollama_install.sh
sed -i.bak -e '/TEMP_DIR=/c TEMP_DIR=/tmp' -e '/curl --fail.*-o/s/^/#/' ollama_install.sh
## 将 ollama-linux-amd64放到tmp下，并重命名为ollama
mv ollama-linux-amd64 /tmp/ollama
## 运行ollama_install.sh即可
sh ollama_install.sh
## 查看ollama状态
systemctl status ollama
```

```
root@localhost:~/mp_mount/llm# sh ollama_install.sh
>>> Downloading ollama...
>>> Installing ollama to /usr/local/bin...
>>> Creating ollama user...
>>> Adding ollama user to render group...
>>> Adding ollama user to video group...
>>> Adding current user to ollama group...
>>> Creating ollama systemd service...
>>> Enabling and starting ollama service...
Created symlink /etc/systemd/system/default.target.wants/ollama.service → /etc/systemd/system/ollama.service.
>>> The Ollama API is now available at 127.0.0.1:11434.
>>> Install complete. Run "ollama" from the command line.
```

此时访问`127.0.0.1:11434`显示`Ollama is running`，则部署成功。

容器形式:

```bash
docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```


## 2. 选择模型

https://ollama.com/ 官网右上角 model

![select-model](../images/local-llm-deploy/select-model.png)



这里模型我们选择阿里的千问2，复制命令`ollama run qwen2`在linux上运行即可，下载时间会有点长，耐心等待。

![open-webui](../images/local-llm-deploy/qwen2.png)

## 3. 安装open-webui

https://github.com/open-webui/open-webui

通过容器化部署。

```bash
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

http://127.0.0.1:3000 访问，需要注册，随便填写即可。

登录后效果如图：

![open-webui](../images/local-llm-deploy/open-webui.png)

至此，我们就成功部署了一个由CPU提供算力的大语言模型。

由于CPU和GPU设计结构的不同，CPU在处理大模型语言时效果并不理想。

根据前面部署完后，使用时会发现模型给出回复的时候反应很慢，且是一个字一个字蹦出来的。

然而GPU驱动大模型能够显著提高其反应速度，回复也是一段一段显示。

下面容器化部署一并介绍如何使用GPU提供算力。

## 4. 容器化部署

使用Nvidia显卡需要下载驱动

> Nvidia 官网
>
> https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

```bash
## 配置apt包仓库地址
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
    
## 更新仓库包列表
sudo apt-get update

## 安装工具
sudo apt-get install -y nvidia-container-toolkit

## 可使用该命令查看gpu
nvidia-smi
## 查看显卡列表
nvidia-smi -L
## 在使用过程中可观察gpu运行情况变化
watch -n 1 nvidia-smi
# 或
nvidia-smi -l

## 运行由gpu驱动的ollama，将大模型存放路径持久化，--gpus all将所有gpu设备加入容器
docker run --gpus all -d -v /opt/ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama

## 下载大模型
docker exec -it ollama ollama pull qwen2:7b
```

**其他包管理工具配置**

```bash
## yum 地址
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
  sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
  
## zypper地址
sudo zypper ar https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo
sudo zypper --gpg-auto-import-keys install -y nvidia-container-toolkit
```

## Others

### mac 环境下部署

Docker Desktop 在 macOS 上运行时，会通过一个轻量级的虚拟机（如 colima 或基于 HyperKit 的虚拟机）来运行 Docker 容器。这种方式确保容器运行在一个 Linux 内核上，而不是 macOS 的内核。因此，/var/lib/docker 实际上存在于虚拟机内，而不是 macOS 本地文件系统中。

进入 Docker Desktop 的虚拟机

```bash
docker context use default
docker run --rm -it --privileged --pid=host justincormack/nsenter1
```

数据位置通常在:  `~/Library/Containers/com.docker.docker/Data/vms/0/`

调整 Docker Desktop 虚拟机的磁盘大小:
`settings -> resuorces -> advanced -> Disk usage limit`
