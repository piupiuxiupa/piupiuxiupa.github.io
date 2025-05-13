---
title: 搭建Docker本地仓库
date: 2024-08-11 15:38:32
tags: docker
categories: Posts
---

# 搭建Docker本地仓库

前提
- 安装 docker
- 熟悉 docker环境及基本操作

> 二进制包部署：https://download.docker.com/linux/static/stable/x86_64/

## 容器化部署

```bash
# 联网环境下
# 直接运行 registry镜像，本地没有会自动拉取
docker run -d -p 5000:5000 --name registry registry:2

# 内网环境
# 下载对应镜像 tar包传入内网环境
# 只有用save的镜像才能load导入，否则会报错
docker save registry:2 -o registry.tar
docker load -i registry.tar

# 也可将运行容器导出
docker export 容器ID > registry.tar
# 将打包容器导入
# 用export导出的容器，需要用import导入
docker import registry.tar localhost/registry:2

## 验证是否安装成功
## 返回 {}表示成功
curl -s http://localhost:5000/v2/
```

## 推送镜像到本地仓库

```bash
## 修改镜像tag为本地仓库地址
docker tag $image localhost:5000/$image

## 推送到本地仓库
docker push localhost:5000/$image

## 拉取本地仓库镜像
docker pull localhost:5000/$image
```

## 持久化存储镜像
```bash
docker run -d -p 5000:5000 --name registry -v /home/registry:/var/lib/registry registry:2
```

## 配置本地仓库HTTPS

可以选择生成自签名证书或使用受信任的证书颁发机构 (CA) 签发的证书。

### 生成自签名证书
- `-newkey rsa:4096`：生成一个新的 RSA 私钥和证书请求，并使用 4096 位的密钥长度
- `-nodes`：指定不对生成的私钥进行加密。如果不使用这个选项，需要为私钥设置一个密码。对于自动化使用，通常不设置密码。
- `-sha256`：使用 SHA-256 哈希算法来生成证书。
- `-keyout certs/domain.key`：指定生成的私钥文件的输出路径。
- `-x509`：表示生成自签名的 X.509 证书。X.509 是一种标准格式，用于公钥证书。
- `-days 365`：设置证书的有效期为 365 天，过期后需要更新。
- `-out certs/domain.crt`：指定生成的自签名证书文件的输出路径。

```bash
mkdir -p /etc/certs

openssl req -newkey rsa:4096 -nodes -sha256 -keyout /etc/certs/domain.key -x509 -days 365 -out /etc/certs/domain.crt
## 运行完会提示需要输入一些信息
## Common Name是最重要的一个
# 需要填写服务器的完全限定域名 (FQDN) 或 IP 地址。
# 如果为 Registry 配置 HTTPS，必须匹配客户端将用来访问服务器的域名或 IP 地址
# 比如本地localhost或者127.0.0.1
```

### registry 配置HTTPS

- `REGISTRY_HTTP_ADDR=0.0.0.0:443`：配置 Registry 监听 HTTPS 请求
- `REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt`：指定 SSL 证书路径
- `REGISTRY_HTTP_TLS_KEY=/certs/domain.key`：指定 SSL 私钥路径

```bash
docker run -d \
  -p 443:443 \
  --name registry \
  -v /etc/certs:/certs \
  -v /home/registry:/var/lib/registry \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
```

### 配置 Docker 客户端信任自签名证书

如果使用的是自签名证书，Docker 客户端默认不会信任你生成的证书。

需要将证书添加到 Docker 的信任列表。

```bash
## 之前生成证书 common name填的 localhost，因此在此处创建localhost
mkdir -p /etc/docker/certs.d/localhost
cp /etc/certs/domain.crt /etc/docker/certs.d/localhost/ca.crt
```

### 配置客户端的 insecure-registries

如果不想配置证书，可以将 Registry 配置为 insecure-registry，但不推荐在生产环境中使用。

```json
{
  "insecure-registries" : ["localhost"]
}
```
重启docker即可生效

```bash
curl -sSL -k https://localhost:443/v2/
```

### 使用反向代理

可以使用反向代理来处理 HTTPS，而 Registry继续使用 HTTP

```conf
server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate /etc/nginx/certs/domain.crt;
    ssl_certificate_key /etc/nginx/certs/domain.key;

    location / {
        proxy_pass http://localhost:5000;
        # 容器化部署时改为容器名 
        proxy_pass http://registry:5000;
    }
}
```
**容器化 nginx反向代理**
```bash
## 创建一个 bridge网络
docker create network proxy_network
## 启动时 nginx和 registry两个容器都指定 proxy_network 网络
docker run -d --name nginx --network proxy_network -p 443:443 -v ~/nginx.conf:/etc/nginx/nginx.conf -v /etc/certs/:/etc/nginx/certs nginx:latest
```


## 设置仓库登陆认证

### 安装 htpasswd
```bash
sudo apt-get install apache2-utils
sudo yum install httpd-tools
```

### 创建认证文件
```bash
mkdir /etc/auth
htpasswd -Bc /etc/auth/htpasswd myuser
## -B：使用 bcrypt 加密算法
## -c：创建一个新的文件
## /etc/auth/htpasswd：保存用户名和密码的文件路径。

## 如果需要添加更多用户，不加 -c
htpasswd -B /etc/auth/htpasswd anotheruser
```
### 配置 Registry登陆认证

```bash
docker run -d \
  -p 5000:5000 \
  --name registry \
  -v /etc/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  -v /home/registry:/var/lib/registry \
  registry:2
# REGISTRY_AUTH=htpasswd：启用 htpasswd 认证
# REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm：设置认证领域（可以随意设定）
# REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd：指定 htpasswd 文件的路径。

## 验证
docker login localhost:5000
```


## 通过api查看本地镜像

> api 返回的信息为 json格式，可以配合linux的 `jq`命令进行筛选查看

```bash
## 获取所有镜像的名称
curl -sSL -k http://localhost:5000/v2/_catalog | jq

## 返回输出
{
  "repositories": [
    "my-image1",
    "my-image2"
  ]
}

## 设置认证后需要修改为如下命令，配合 jq使用提取信息 
curl -sSL -k -u $username:$passwd http://127.0.0.1:5000/v2/_catalog |jq -r ".repositories[]"

## 获取每个镜像的标签
curl http://localhost:5000/v2/$image/tags/list
curl -sSL -k -u $username:$passwd http://127.0.0.1:5000/v2/$image/tags/list

## 镜像的新版本会靠前显示
{
  "name": "my-image1",
  "tags": [
    "latest",
    "v2.0",
    "v1.0"
  ]
}

## 获取 digest信息
curl -u $username:$passwd -sSL -H "Accept: application/vnd.docker.distribution.manifest.v2+json" http://127.0.0.1:5000/v2/$image/manifests/$tag | jq -r ".config.digest"

## 通过delete方法删除对应镜像的 tag
curl -sSLf -k -X DELETE -u $username:$passwd http://127.0.0.1:5000/v2/$image/manifests/$digest

## 清理未被引用的blobs
registry garbage-collect -m /etc/docker/registry/config.yml
```
