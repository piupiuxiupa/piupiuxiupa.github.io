---
title: 本地大模型部署
date: 2024-08-04 18:59:10
tags: tools
---

# 本地大模型部署

> Windows根据官网下载安装即可

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

至此，我们就成功部署了一个由cpu提供算力的大语言模型。