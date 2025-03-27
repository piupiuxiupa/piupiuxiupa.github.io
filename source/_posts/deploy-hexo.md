---
title: Hexo部署GitHub个人博客
date: 2024-08-05 10:42:50
tags: tools
categories: Posts
---

# Hexo部署GitHub个人博客

## 1. 安装Hexo

安装Hexo需要

- nodejs
- npm

```bash
# 注意nodejs版本
# 需要 ubuntu noble版本
apt install nodejs npm
# npm 下载慢需要跟换国内源，或者添加代理
npm install -g hexo
```

## 2. Hexo 常用命令

```bash
# 需要一点时间
hexo init $proj_name

## 新建post
hexo new "My First Blog"

## 启动本地server
hexo server

## 生成页面并部署到github
hexo generate && hexo deploy
## 可简写
hexo g
hexo d
```

## 3. 配置 GitHub

1. 新建一个github项目

2. 点进项目找到Settings -> Pages 进行配置 GitHub Pages

   ![page](../images/deploy-hexo/page.png)

```bash
# 将项目clone下来
# 此处最好配置ssh类型, 不然后面每次部署推送都需要输入账号和token，且ssh不容易出现某些网络问题
git clone $url
# 创建一个分支用来存储hexo源代码
git checkout -b hexo
```

**配置SSH方式**

```bash
# 生成sshkey
ssh-keygen -t rsa
## 将公钥复制到github的项目settings -> Deploy keys里
cat ~/.ssh/id_rsa.pub
```
> GitHub从2021年8月13号开始不能用密码推送，需要用token。
>
> token需要在右上角个人Settings -> [Developer Settings](https://github.com/settings/apps) -> [Personal access tokens](https://github.com/settings/tokens)里生成。
>
> 注意保管，丢失了需要重新生成。


## 4. 配置 Hexo

```bash
## 查看目录结构
tree -L 1 
## 输出如下
MyBlog/
├── _config.landscape.yml ## 主题配置文件，默认主题landscape，主题可在 https://hexo.io/themes/ 官网自行寻找
├── _config.yml ## hexo配置文件
├── node_modules
├── package-lock.json
├── package.json
├── scaffolds
├── source ## post存放目录，可在同级建立image存放图片
└── themes
```

此处`_config.yml`配置自行修改，注意url填写你的 GitHub Pages地址。

![config](../images/deploy-hexo/config.png)

在`_config.yml`最后有deploy的配置选项，默认如图：

![](../images/deploy-hexo/deploy-config.png)

修改为：

![](../images/deploy-hexo/deploy-git.png)

branch填写你需要部署的分支，我这里选择master；

repo 填写git地址，可以是https形式，也可以是ssh形式。

## 5. 部署&推送到仓库

```bash
## hexo 会生成一个 .deploy_git
## 所以我们将这个文件写入 .gitignore
echo ".deploy_git/" >> .gitignore
## 如果之前有过推送需要删掉
git rm -r --cached .deploy_git

## 生成页面并部署，因为配置过git仓地址，会自动触发github action 进行部署
hexo g && hexo d

## 将源文件推送到hexo分支
git push origin hexo

```

