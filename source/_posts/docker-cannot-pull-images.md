---
title: Docker cannot pull images
date: 2025-03-07 09:58:20
tags: docker
categories: Posts
---

# Mac Docker 镜像网络问题

国内环境无法拉取docker镜像，网上有很多方法。

有修改 docker镜像源的，有往配置文件加代理或者加dns的。

个人常用方法是用喵喵，以前也只需要挂喵喵即可，最近发现只用它也没用了。

不知道是因为换了mac有什么不同，还是因为某些wall的限制加强了。

然后尝试修改了dns解析，成功了，dns地址使用

## Mac 终端配置代理

```bash
alias proxy='export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890'
alias unproxy='unset https_proxy http_proxy all_proxy'
```

## Mac修改DNS

- 可以通过mac界面的网络来修改
- 也可以通过命令行修改

个人一般需要修改的场景只在terminal中，所以将命令做成alias来快捷使用。

修改dns命令如下：

```bash
# 查看网络接口
networksetup -listallnetworkservice

networksetup -setdnsservers Wi-Fi 8.8.8.8 114.114.114.114

# 查看wi-fi接口的dns配置
networksetup -getdnsservers Wi-Fi

# 或用scutil查看dns配置
scutil --dns

# 清空dns配置，即恢复默认
networksetup -setdnsservers Wi-Fi empty

# 刷新dns缓存
dscacheutil -flushcache
```

## DNS推荐

谷歌DNS: 8.8.8.8  -- 个人尝试下来比较推荐

腾讯DNS : 119.29.29.29

AliDNS 阿里
首选： 223.5.5.5 备用： 223.6.6.6

114 DNS
常规公共 DNS (干净无劫持)

首选： 114.114.114.114

备选： 114.114.115.115

拦截钓鱼病毒木马网站 (保护上网安全)

首选： 114.114.114.119

备用： 114.114.115.119

拦截色情网站 (保护儿童)

首选： 114.114.114.110

备用： 114.114.115.110
