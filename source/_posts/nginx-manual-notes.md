---
title: Nginx HTTP 核心模块 (ngx_http_core_module) 完全指南
date: 2026-04-20 14:54:04
tags: tools
categories: Notes
---

> 本文档基于 nginx 官方文档 https://nginx.org/en/docs/http/ngx_http_core_module.html 编写，涵盖该模块所有指令和内嵌变量的详细说明。

## 目录

- [概述](#概述)
- [配置上下文说明](#配置上下文说明)
- [指令详解](#指令详解)
  - [服务器与监听](#服务器与监听)
  - [请求处理与路由](#请求处理与路由)
  - [客户端请求控制](#客户端请求控制)
  - [连接与超时管理](#连接与超时管理)
  - [文件处理与传输](#文件处理与传输)
  - [重定向与响应控制](#重定向与响应控制)
  - [路径与安全](#路径与安全)
  - [DNS 解析](#dns-解析)
  - [性能调优](#性能调优)
  - [日志与调试](#日志与调试)
  - [子请求与其他](#子请求与其他)
- [内嵌变量参考](#内嵌变量参考)
- [典型配置示例](#典型配置示例)
- [最佳实践总结](#最佳实践总结)

---

## 概述

ngx_http_core_module 是 nginx 的基础 HTTP 模块。它不是可选插件，而是 nginx 处理 HTTP 流量的根基。你在 nginx.conf 里写的绝大多数配置，都来自这个模块。

它提供的能力覆盖了 HTTP 服务器的全部核心功能：

- **监听端口**：通过 `listen` 指令绑定 IP 地址和端口，接收客户端连接
- **路由请求**：通过 `location` 指令把不同的 URI 路径分发给不同的处理逻辑
- **静态文件服务**：直接从磁盘读取文件并返回给客户端，这是 nginx 最常见的用途
- **连接管理**：控制超时、连接数、keepalive 行为
- **请求控制**：限制客户端请求体大小、允许的 HTTP 方法、缓冲区等
- **重定向与响应**：配置错误页面、重定向规则、返回自定义状态码
- **文件传输优化**：sendfile、aio、直接 I/O 等高性能文件传输机制

几乎所有 nginx 配置文件都会大量使用这个模块的指令。无论你是搭建静态文件服务器、配置反向代理，还是做负载均衡，ngx_http_core_module 都是最底层、最关键的依赖。

本指南覆盖该模块全部 82 条指令和 55 个以上的内嵌变量。每条指令都标注了可用的配置上下文、默认值、适用版本，并附带使用说明和注意事项。

---

## 配置上下文说明

nginx 的配置文件采用嵌套的块结构。指令不是在任意位置都能生效的，每条指令都有自己的"作用域"，也就是它允许出现的配置上下文（context）。

理解上下文层级是正确编写 nginx 配置的前提。

### 上下文层级

```
main (全局)
└── http { } (HTTP 全局)
    └── server { } (虚拟服务器)
        └── location { } (请求匹配)
            └── if (condition) { } (条件判断)
```

### 各级上下文详解

**main（全局上下文）**

位于配置文件最外层，不属于任何 `{ }` 块。这里的指令影响 nginx 整体行为，比如 worker 进程数、错误日志路径、events 块的配置。 ngx_http_core_module 的指令一般不出现在这个层级（它们属于 http 块内部），但部分跨模块的全局参数（如 `worker_processes`）在这里设置。

**http（HTTP 全局上下文）**

在 `http { }` 块内部。这里的指令对所有 HTTP 流量生效，是所有 server 块和 location 块的公共配置层。你可以在 http 层设置默认的超时时间、日志格式、文件路径等，然后让下层的 server 或 location 覆盖特定值。

**server（虚拟服务器上下文）**

在 `server { }` 块内部。每个 server 块定义一个虚拟主机（virtual host），通过 `listen` 绑定地址端口，通过 `server_name` 匹配域名。同一个 nginx 实例可以运行多个 server 块，各自独立处理不同域名的请求。

**location（请求匹配上下文）**

在 `location { }` 块内部。这是最细粒度的路由控制。location 根据请求 URI 进行匹配，匹配成功后，该块内的指令只对匹配到的请求生效。location 块可以嵌套（某些类型的 location 内部再放 location）。

**if in location（location 内的条件上下文）**

在 location 块内部的 `if (condition) { }` 块中。用于根据条件动态改变处理逻辑。nginx 的 `if` 有很多陷阱，被称为 "if is evil"，建议仅在简单条件下使用（如检查变量是否存在、匹配正则等），避免在其中做复杂的重写逻辑。

### 完整示例

下面这个例子展示了所有上下文层级嵌套在一起的样子：

```nginx
# === main 上下文（最外层，不属于任何块） ===
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;

events {
    worker_connections 1024;
}

http {
    # === http 上下文 ===
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    # 日志格式定义
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';

    access_log /var/log/nginx/access.log main;

    # --- server 上下文：第一个虚拟主机 ---
    server {
        listen       80;
        server_name  example.com www.example.com;

        # location 上下文：处理根路径
        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        # location 上下文：处理图片请求
        location /images/ {
            root /data/images;

            # if in location 上下文
            if ($args ~ "download=1") {
                add_header Content-Disposition 'attachment';
            }
        }

        # location 上下文：错误页面
        error_page 404 /404.html;
        location = /404.html {
            internal;
        }
    }

    # --- server 上下文：第二个虚拟主机 ---
    server {
        listen       80;
        server_name  api.example.com;

        location / {
            proxy_pass http://127.0.0.1:8080;
        }
    }
}
```

### 上下文继承规则

理解 nginx 的继承机制很重要：

1. **向下继承**：http 层设置的指令值会自动传递给所有 server 块，server 层的值会传递给所有 location 块。这和 CSS 的层叠规则类似，子级可以覆盖父级的设置。

2. **就近原则**：如果多层都设置了同一条指令，最内层的值生效。比如 http 层设了 `client_max_body_size 1m`，但某个 location 里设了 `client_max_body_size 10m`，那么匹配到该 location 的请求会使用 10m 的限制。

3. **数组类指令特殊**：有些指令可以多次出现（如 `access_log`、`error_page`），它们在子上下文中一旦出现，就不会继承父级的同名指令，而是在当前层级重新定义。如果你需要"追加"而不是"覆盖"，要手动把父级的内容再写一遍。

4. **`proxy_set_header` 等指令的陷阱**：这类指令在 location 层出现时，会完全覆盖 http/server 层的定义，而不是追加。如果你在 server 层设了 `proxy_set_header X-Real-IP $remote_addr;`，又在 location 层设了 `proxy_set_header X-Custom value;`，那么 `X-Real-IP` 会在该 location 中丢失。解决办法是在 location 层把两条都写上。
## 服务器与监听

### http

- 语法: `http { ... }`
- 默认值: —
- 适用上下文: main
- 说明: 提供HTTP服务器指令的配置文件上下文。这是所有HTTP相关配置的顶层容器。
- 示例:
```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    keepalive_timeout  65;
    
    server {
        listen 80;
        server_name localhost;
    }
}
```

### server

- 语法: `server { ... }`
- 默认值: —
- 适用上下文: http
- 说明: 设置虚拟服务器的配置。nginx不区分基于IP和基于名称（域名）的虚拟服务器，统一由server块定义。server_name指令用于指定域名。
- 示例:
```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/example;
}
```

### listen

- 语法: `listen address[:port] [default_server] [ssl] [http2 | quic] [proxy_protocol] [setfib=number] [fastopen=number] [backlog=number] [rcvbuf=size] [sndbuf=size] [accept_filter=filter] [deferred] [bind] [ipv6only=on|off] [reuseport] [so_keepalive=on|off|[keepidle]:[keepintvl]:[keepcnt]];`
- 也可以: `listen port [options...];` 或 `listen unix:path [options...];`
- 默认值: `listen *:80 | *:8000;`
- 适用上下文: server
- 说明: 详细说明listen指令的功能，包括：
  - 设置服务器接受请求的IP地址和端口
  - 支持IPv6地址（如 `[::]:80`）
  - 支持UNIX域套接字
  - default_server参数指定该server为默认服务器（当没有匹配到server_name时使用）
  - ssl参数启用HTTPS
  - http2参数启用HTTP/2
  - quic参数启用HTTP/3（QUIC）
  - proxy_protocol参数启用PROXY协议
  - backlog设置监听队列的最大长度
  - rcvbuf/sndbuf设置socket的接收/发送缓冲区大小
  - bind强制对指定地址进行bind()调用
  - reuseport允许多个worker进程绑定同一端口
  - deferred在Linux上使用TCP_DEFER_ACCEPT
  - so_keepalive配置TCP keepalive参数
  - 只能指定一个default_server
- 示例:
```nginx
server {
    # 监听所有接口的80端口
    listen 80;
    
    # 监听特定IP和端口，设为默认服务器
    listen 192.168.1.1:80 default_server;
    
    # HTTPS + HTTP/2
    listen 443 ssl http2;
    
    # IPv6
    listen [::]:80;
    
    # UNIX域套接字
    listen unix:/var/run/nginx.sock;
    
    # 带backlog和buffer设置
    listen 443 ssl backlog=1024 rcvbuf=32k sndbuf=32k;
}
```

### server_name

- 语法: `server_name name ...;`
- 默认值: `server_name "";`
- 适用上下文: server
- 说明: 详细说明：
  - 设置虚拟服务器的名称
  - 第一个名称成为主服务器名称
  - 支持通配符：`*.example.com`、`www.*`、`*.example.*`（仅在开头或结尾使用*）
  - 支持正则表达式：以~开头，如 `~^(www\.)?(.+)$`
  - 特殊值 `_` 常用作不匹配任何域名的占位符
  - 空字符串""匹配没有Host头的请求
  - 支持捕获组在正则中使用，如 `$1`、`$2`
  - 名称匹配优先级：精确名称 > 最长以*开头的通配符 > 最长以*结尾的通配符 > 第一个匹配的正则表达式
- 示例:
```nginx
server {
    listen 80;
    # 精确匹配
    server_name example.com www.example.com;
}

server {
    listen 80;
    # 通配符匹配
    server_name *.example.com;
}

server {
    listen 80;
    # 正则匹配
    server_name ~^(www\.)?(.+)$;
}

server {
    listen 80;
    # 默认服务器占位符
    server_name _;
}
```
# 请求处理与路由

## location

**语法:** `location [ = | ~ | ~* | ^~ ] uri { ... }` 或 `location @name { ... }`

**默认值:** —

**适用上下文:** server, location

**说明:** 根据请求URI设置配置。详细解释匹配规则：

- `= /exact/path` — 精确匹配，优先级最高，匹配后立即处理
- `^~ /prefix/path` — 前缀匹配，如果匹配则不再检查正则
- `~ /regex/` — 区分大小写的正则匹配
- `~* /regex/` — 不区分大小写的正则匹配
- `/prefix/path` — 普通前缀匹配
- `@name` — 命名位置，仅用于内部重定向
- 匹配顺序：精确(=) → 最长前缀(^~) → 正则(按配置文件中的顺序) → 最长前缀

**示例:**

```nginx
server {
    # 精确匹配 - 优先级最高
    location = / {
        proxy_pass http://backend;
    }

    # 前缀匹配 + 不检查正则
    location ^~ /static/ {
        root /var/www;
    }

    # 区分大小写的正则匹配
    location ~ \.php$ {
        fastcgi_pass unix:/tmp/php.sock;
    }

    # 不区分大小写的正则匹配
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 30d;
    }

    # 命名位置（仅内部重定向）
    location @fallback {
        proxy_pass http://backup;
    }

    # 普通前缀匹配
    location /images/ {
        root /var/www;
    }
}
```

## root

**语法:** `root path;`

**默认值:** `root html;`

**适用上下文:** http, server, location, if in location

**说明:** 设置请求的根目录。path值可以包含变量（$document_root和$realpath_root除外）。请求URI会附加到root路径后面。例如，root /data/www + URI /images/1.jpg → 实际路径 /data/www/images/1.jpg

**示例:**

```nginx
server {
    root /var/www/html;

    location /images/ {
        root /data;  # /data/images/...
    }

    location ~ ^/download/(.*)$ {
        root /data;  # /data/download/...
    }
}
```

## alias

**语法:** `alias path;`

**默认值:** —

**适用上下文:** location

**说明:** 定义指定location的路径替换。与root不同，alias会将location匹配的部分替换为指定路径。path值可以包含变量（$document_root和$realpath_root除外）。当location使用正则匹配时，alias必须使用捕获组。注意：alias路径末尾的斜杠很重要。

**示例:**

```nginx
# root vs alias 区别:
# 请求 /i/top.gif
location /i/ {
    root /data/w3;      # 实际路径: /data/w3/i/top.gif
}

location /i/ {
    alias /data/w3/images/;  # 实际路径: /data/w3/images/top.gif
}

# 正则匹配中使用alias
location ~ ^/download/(.+)$ {
    alias /data/files/$1;
}
```

## try_files

**语法:** `try_files file ... uri;` 或 `try_files file ... =code;`

**默认值:** —

**适用上下文:** server, location

**说明:** 按指定顺序检查文件是否存在，使用第一个找到的文件进行处理。如果在最后一个参数中指定了uri，则进行内部重定向。如果最后一个参数是=code，则返回对应的状态码。path可以包含变量。可以使用$fallback引用命名location。

**示例:**

```nginx
location / {
    try_files $uri $uri/ @fallback;
}

location @fallback {
    proxy_pass http://backend;
}

# 另一种写法
location / {
    try_files $uri $uri/ /index.html;
}

# 返回错误码
location / {
    try_files $uri $uri/ =404;
}
```

## error_page

**语法:** `error_page code ... [=[response]] uri;`

**默认值:** —

**适用上下文:** http, server, location, if in location

**说明:** 定义指定错误的URI。允许变量、响应码覆盖（=response）和命名位置。支持URL重定向（如果uri以http/https开头，则客户端被重定向）。`=response`语法允许改变返回的状态码。

**示例:**

```nginx
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;

# 改变状态码
error_page 404 =200 /index.html;

# 使用命名location
error_page 404 @fallback;

# 重定向
error_page 404 https://example.com/not_found;

# 使用变量
error_page 404 /404.php?url=$request_uri;
```

## internal

**语法:** `internal;`

**默认值:** —

**适用上下文:** location

**说明:** 指定给定location只能用于内部请求。外部请求返回404。内部请求包括：error_page重定向、index指令、try_files、X-Accel-Redirect响应头、rewrite、子请求。

**示例:**

```nginx
error_page 404 /404.html;

location /404.html {
    internal;
    root /var/www/errors;
}
```

## types

**语法:** `types { ... }`

**默认值:** `types { text/html html; image/gif gif; image/jpeg jpg; }` (以及更多默认MIME类型)

**适用上下文:** http, server, location

**说明:** 将文件扩展名映射到响应的MIME类型。扩展名不区分大小写。通常通过include mime.types加载。一个扩展名可以映射到多个MIME类型。

**示例:**

```nginx
types {
    text/html             html htm shtml;
    text/css              css;
    application/javascript js;
    image/jpeg            jpeg jpg;
    application/json      json;
}

# 禁用所有MIME类型
location /download/ {
    types { }
    default_type application/octet-stream;
}
```

## default_type

**语法:** `default_type mime-type;`

**默认值:** `default_type text/plain;`

**适用上下文:** http, server, location

**说明:** 定义响应的默认MIME类型。当types指令中未匹配到文件扩展名时，使用此默认值。

**示例:**

```nginx
default_type application/octet-stream;

location /api/ {
    default_type application/json;
}

location /download/ {
    types { }
    default_type application/octet-stream;
}
```
## 客户端请求控制

### client_max_body_size

**语法：** `client_max_body_size size;`

**默认值：** `1m`

**适用上下文：** `http`、`server`、`location`

设置客户端请求体的最大允许大小。当请求体超过这个值时，nginx 会返回 413 (Request Entity Too Large) 错误给客户端。将值设为 `0` 可以禁用这项检查，不对请求体大小做限制。

这个指令最常用于控制文件上传的大小上限。比如你的站点允许用户上传头像、附件，就需要根据业务需求调整这个值。

```nginx
# 允许上传最大 20MB 的文件
server {
    client_max_body_size 20m;
}

# 图片上传接口允许 5MB，视频上传接口允许 500MB
location /upload/image {
    client_max_body_size 5m;
}

location /upload/video {
    client_max_body_size 500m;
}
```

### client_body_buffer_size

**语法：** `client_body_buffer_size size;`

**默认值：** `8k`（x86/32 位系统）| `16k`（64 位系统）

**适用上下文：** `http`、`server`、`location`

设置读取客户端请求体时使用的缓冲区大小。如果请求体超过这个缓冲区大小，整个请求体或者其中一部分会被写入临时文件。

缓冲区调大可以减少磁盘 I/O，但会占用更多内存。实际使用中需要根据请求体大小的分布来权衡。

```nginx
# 把缓冲区调到 128K，减少小请求体写入临时文件的频率
location /api/ {
    client_body_buffer_size 128k;
}
```

### client_body_in_file_only

**语法：** `client_body_in_file_only on | clean | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

决定 nginx 是否将整个客户端请求体保存到文件中。三个值的含义：

- `on`：始终将请求体写入文件，请求结束后保留临时文件（适合调试）
- `clean`：始终将请求体写入文件，请求处理完成后删除临时文件
- `off`：仅在请求体超过缓冲区大小时才写入文件（默认行为）

```nginx
# 调试场景：保留所有请求体到文件用于分析
location /debug/ {
    client_body_in_file_only on;
}

# 确保请求体始终写入文件，但处理完自动清理
location /upload/ {
    client_body_in_file_only clean;
}
```

### client_body_in_single_buffer

**语法：** `client_body_in_single_buffer on | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

决定是否将整个客户端请求体保存在单个连续的缓冲区中。默认情况下请求体可能分散在多个缓冲区里。当你需要使用 `$request_body` 变量读取请求体内容时，建议启用这个选项，因为该变量只在请求体存在单个缓冲区中时才能正确获取完整数据。

```nginx
# 需要读取 $request_body 变量时启用
location /webhook {
    client_body_in_single_buffer on;
    # 后续可以通过 $request_body 获取完整请求体
}
```

### client_body_temp_path

**语法：** `client_body_temp_path path [level1 [level2 [level3]]];`

**默认值：** `client_body_temp`

**适用上下文：** `http`、`server`、`location`

定义存储客户端请求体临时文件的目录。支持最多三级子目录层级，子目录名由临时文件名的十六进制哈希值生成。使用多级子目录可以避免单个目录下文件过多带来的性能问题。

```nginx
# 自定义临时文件目录，使用两级子目录
http {
    client_body_temp_path /var/tmp/nginx/client_body 2 2;
}

# 不同 location 可以使用不同的临时目录
location /upload/large {
    client_body_temp_path /data/tmp/upload 1 2 2;
}
```

### client_body_timeout

**语法：** `client_body_timeout time;`

**默认值：** `60s`

**适用上下文：** `http`、`server`、`location`

定义读取客户端请求体的超时时间。这个超时不是针对整个请求体的传输时间，而是仅针对两次连续读操作之间的间隔。如果客户端在传输请求体时两次读操作之间的等待时间超过设定值，nginx 会返回 408 (Request Time-out) 错误并关闭连接。

网络状况较差或客户端上传大文件时，可能需要适当增大这个值。

```nginx
# 上传大文件时给予更长的超时
location /upload/ {
    client_max_body_size 200m;
    client_body_timeout 300s;
}
```

### client_header_buffer_size

**语法：** `client_header_buffer_size size;`

**默认值：** `1k`

**适用上下文：** `http`、`server`

设置读取客户端请求头的缓冲区大小。对于绝大多数常规请求，1K 的缓冲区已经足够。当请求头超出这个大小时，nginx 会使用 `large_client_header_buffers` 指令分配的更大缓冲区来处理。

如果业务场景中客户端会携带大量 Cookie 或自定义头部，可以适当调大这个值。

```nginx
# 应对大型 Cookie 或多个自定义头部的场景
server {
    client_header_buffer_size 4k;
}
```

### client_header_timeout

**语法：** `client_header_timeout time;`

**默认值：** `60s`

**适用上下文：** `http`、`server`

定义读取客户端请求头的超时时间。如果客户端没有在规定时间内把完整的请求头传输完毕，nginx 会返回 408 (Request Time-out) 错误并关闭连接。

这个值主要用于防御慢速攻击（Slowloris），攻击者通过极慢地发送请求头来长期占用连接资源。

```nginx
# 适当缩短超时以防御慢速攻击
http {
    client_header_timeout 15s;
}
```

### large_client_header_buffers

**语法：** `large_client_header_buffers number size;`

**默认值：** `4 8k`

**适用上下文：** `http`、`server`

设置用于读取大型客户端请求头的缓冲区的最大数量和大小。当请求头超出 `client_header_buffer_size` 的容量时，nginx 会从这里分配更大的缓冲区。

- 如果请求行（Request Line）超过一个缓冲区的大小，返回 414 (Request-URI Too Large)
- 如果请求头字段超过一个缓冲区的大小，返回 400 (Bad Request)

```nginx
# 增大缓冲区以应对超长 Cookie 的场景
server {
    large_client_header_buffers 8 16k;
}
```

### limit_except

**语法：** `limit_except method ... { ... }`

**默认值：** —

**适用上下文：** `location`

限制 location 块内允许使用的 HTTP 方法。支持的方法包括：`GET`、`HEAD`、`POST`、`PUT`、`DELETE`、`MKCOL`、`COPY`、`MOVE`、`OPTIONS`、`PROPFIND`、`PROPPATCH`、`LOCK`、`UNLOCK`、`PATCH`。

列出的方法为允许的方法，未列出的方法会被限制。允许 `GET` 方法时会自动允许 `HEAD` 方法。这个指令通常和访问控制模块（如 `deny`、`allow`）配合使用。

```nginx
# 只允许 GET 和 POST，其余方法全部拒绝
location /api/ {
    limit_except GET POST {
        deny all;
    }
}

# 静态资源只允许 GET 和 HEAD
location /static/ {
    limit_except GET {
        deny all;
    }
}
```

### limit_rate

**语法：** `limit_rate rate;`

**默认值：** `0`

**适用上下文：** `http`、`server`、`location`、`if in location`

限制向客户端传输响应的速率，单位为字节/秒。设为 `0` 表示不限速。从 1.17.0 版本开始支持使用变量动态设置限速值。这个指令按请求生效，不同请求之间互不影响。

```nginx
# 下载目录限速 100KB/s
location /download/ {
    limit_rate 100k;
}

# 基于变量动态限速
server {
    if ($slow) {
        set $limit_rate 50k;
    }
}

# 根据客户端条件设置不同速率
location /video/ {
    set $limit_rate 500k;
    if ($arg_quality = "low") {
        set $limit_rate 100k;
    }
}
```

### limit_rate_after

**语法：** `limit_rate_after size;`

**默认值：** `0`

**适用上下文：** `http`、`server`、`location`、`if in location`

设置开始限速之前先传输的数据量。简单说就是前 N 字节不限速，传完之后再按 `limit_rate` 的设定限速。从 1.17.0 版本开始支持使用变量。

这在视频点播、文件下载等场景中很实用：先快速传输开头部分让用户尽快看到内容或确认文件有效，然后再限速以控制带宽。

```nginx
# 视频：前 1MB 不限速（快速起播），之后限速 100KB/s
location /video/ {
    limit_rate_after 1m;
    limit_rate 100k;
}

# 文件下载：前 512KB 不限速，之后限速 50KB/s
location /files/ {
    limit_rate_after 512k;
    limit_rate 50k;
}
```

### max_ranges

**语法：** `max_ranges number;`

**默认值：** —（不限制）

**适用上下文：** `http`、`server`、`location`

限制字节范围请求（Range Request）中允许的最大范围数。当客户端请求的范围数超过这个限制时，nginx 会忽略所有范围请求，按普通请求返回完整内容。默认不限制范围数。

通过限制范围数可以防止客户端发起过多的小范围请求来消耗服务器资源。

```nginx
# 最多允许 3 个范围
location /download/ {
    max_ranges 3;
}

# 完全禁用范围请求
location /secure-files/ {
    max_ranges 0;
}
```

### max_headers

**语法：** `max_headers number;`

**默认值：** `1000`

**适用上下文：** `http`、`server`（v1.29.8+）

设置单个请求中允许的最大头部行数。当请求的头部行数达到这个限制时，nginx 会返回 400 (Bad Request) 错误。这个指令从 1.29.8 版本开始提供。

调低这个值可以防止客户端发送海量头部来消耗服务器内存。

```nginx
# 限制最多 50 个头部行
server {
    max_headers 50;
}
```
## 连接与超时管理

### keepalive_timeout

**语法：** `keepalive_timeout timeout [header_timeout];`

**默认值：** `75s`

**适用上下文：** `http`、`server`、`location`

设置 keep-alive 连接在服务器端保持打开的超时时间。设为 `0` 则禁用 keep-alive 功能。可选的第二参数用于设置响应头 `Keep-Alive: timeout=time` 的值，让客户端知道服务器端的超时策略。两个参数可以不同，服务器端实际超时用第一个值，客户端感知到的是第二个值。

```nginx
# 服务器端超时 65 秒，响应头中显示 60 秒
keepalive_timeout 65s 60s;

# 禁用 keep-alive
keepalive_timeout 0;
```

### keepalive_requests

**语法：** `keepalive_requests number;`

**默认值：** `1000`

**适用上下文：** `http`、`server`、`location`

设置单个 keep-alive 连接上可以服务的最大请求数。连接达到这个上限后会被关闭，客户端需要建立新连接才能继续请求。这个限制能防止单个长连接占用资源过久。在高并发场景下可以适当调大，但不要设得过高，否则内存泄漏或连接堆积的风险会增加。

```nginx
# 允许每个长连接处理最多 2000 个请求
keepalive_requests 2000;
```

### keepalive_time

**语法：** `keepalive_time time;`

**默认值：** `1h`

**适用上下文：** `http`、`server`、`location`

**版本要求：** 1.19.10+

限制一个 keep-alive 连接处理请求的最大存活时间。连接存在时间超过这个值后，在处理完当前请求后关闭。注意这和 `keepalive_timeout` 不同：`keepalive_timeout` 控制的是空闲等待时间，`keepalive_time` 控制的是连接从建立到关闭的总生存时间。

```nginx
# 连接最长存活 2 小时，不管期间处理了多少请求
keepalive_time 2h;
```

### keepalive_disable

**语法：** `keepalive_disable none | browser ...;`

**默认值：** `msie6`

**适用上下文：** `http`、`server`、`location`

针对特定浏览器禁用 keep-alive 连接。某些旧版浏览器对 keep-alive 的实现存在问题，需要特殊处理：

- `msie6`：禁用旧版 MSIE（6.0 及更早版本）在发送 POST 请求后的 keep-alive 连接
- `safari`：禁用 macOS 上类似 Safari 的浏览器的 keep-alive 连接
- `none`：对所有浏览器都启用 keep-alive

可以同时指定多个浏览器。

```nginx
# 同时禁用 MSIE6 和 Safari 的 keep-alive
keepalive_disable msie6 safari;

# 对所有浏览器启用 keep-alive
keepalive_disable none;
```

### keepalive_min_timeout

**语法：** `keepalive_min_timeout timeout;`

**默认值：** `0`

**适用上下文：** `http`、`server`、`location`

**版本要求：** 1.27.4+

设置 keep-alive 连接在服务器端不被关闭的最短超时时间。这个值主要用于连接复用或优雅关闭场景。设为 `0` 表示不限制。当工作进程正在进行优雅关闭（graceful shutdown）时，keep-alive 连接如果在 `keepalive_min_timeout` 时间内还有剩余存活期，就不会被立即关闭，从而允许客户端在现有连接上继续发送请求。

```nginx
# 保证 keep-alive 连接至少存活 10 秒
keepalive_min_timeout 10s;
```

### send_timeout

**语法：** `send_timeout time;`

**默认值：** `60s`

**适用上下文：** `http`、`server`、`location`

设置向客户端传输响应的超时时间。需要注意，这不是整个响应传输的总超时，而是两次连续写操作之间的超时。如果在这段时间内客户端没有接收任何数据（比如客户端网络拥堵或处理缓慢），nginx 就会关闭连接。

```nginx
# 发送超时设为 30 秒
send_timeout 30s;
```

### lingering_close

**语法：** `lingering_close off | on | always;`

**默认值：** `on`

**适用上下文：** `http`、`server`、`location`

控制 nginx 如何关闭客户端连接，主要影响连接关闭时对客户端额外数据的处理方式：

- `on`：启发式判断。如果客户端还有数据未读取（比如正在上传），nginx 会等待一小段时间把数据读完后关闭
- `always`：无条件等待客户端发送完所有数据后再关闭，确保连接被干净地终止
- `off`：立即关闭连接，不等待任何额外数据。速度快但可能违反协议规范，某些客户端会收到错误提示

一般情况下用默认的 `on` 就行。调试或排查问题时可以用 `always` 确保连接干净关闭。

```nginx
# 对所有连接都等待额外数据处理完毕
lingering_close always;
```

### lingering_time

**语法：** `lingering_time time;`

**默认值：** `30s`

**适用上下文：** `http`、`server`、`location`

当 `lingering_close` 生效时，指定 nginx 处理（读取并忽略）客户端额外数据的最大时间。超过这个时间后，连接会被强制关闭，不管还有没有数据没读完。这是一个总时间上限，防止某些慢速客户端长时间占用连接资源。

```nginx
# 最多花 15 秒处理客户端残留数据
lingering_time 15s;
```

### lingering_timeout

**语法：** `lingering_timeout time;`

**默认值：** `5s`

**适用上下文：** `http`、`server`、`location`

当 `lingering_close` 生效时，指定每次等待客户端数据的最大时间。nginx 会反复执行"等待数据、读取数据、丢弃数据"的循环，每次等待时间不超过 `lingering_timeout`，整个循环的总时间不超过 `lingering_time`。

```nginx
# 每次最多等 3 秒看客户端还有没有数据
lingering_timeout 3s;
```

### reset_timedout_connection

**语法：** `reset_timedout_connection on | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

启用后，超时的连接和使用非标准代码 444 关闭的连接会被重置（发送 RST 包），而不是正常关闭（发送 FIN 包）。重置通过设置套接字的 `SO_LINGER` 选项（超时为 0）实现，这样可以避免连接进入 `TIME_WAIT` 状态，释放资源更快。在连接量很大的服务器上开启这个选项能有效减少 `TIME_WAIT` 状态的连接数。

```nginx
# 重置超时连接，避免 TIME_WAIT 状态
reset_timedout_connection on;
```

---

### 综合配置示例

下面是一个在实际生产环境中使用的 keep-alive 和超时管理的完整配置，涵盖了上述所有指令：

```nginx
http {
    keepalive_timeout 65s 60s;        # 服务器端空闲超时 65 秒，响应头中通知客户端 60 秒
    keepalive_requests 1000;           # 每个长连接最多处理 1000 个请求
    keepalive_time 2h;                 # 连接最长存活 2 小时
    keepalive_disable msie6;           # 禁用 MSIE6 的 keep-alive

    send_timeout 30s;                  # 连续两次写操作间隔超时 30 秒
    lingering_close on;                # 启用优雅关闭，启发式判断
    lingering_time 30s;                # 优雅关闭最多等待 30 秒处理残留数据
    lingering_timeout 5s;              # 每次读取残留数据最多等 5 秒
    reset_timedout_connection on;      # 重置超时连接，减少 TIME_WAIT
}
```
## 文件处理与传输

Nginx在静态文件服务和反向代理场景中，文件传输效率直接影响整体性能。这一节涵盖的指令控制着Nginx如何读取、缓存和发送文件内容，从内核级零拷贝传输到异步I/O，再到文件描述符缓存，每一层都有对应的调优手段。

---

### sendfile

**语法：** `sendfile on | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`、`if in location`

启用或禁用 `sendfile()` 系统调用。传统文件传输需要先把数据从磁盘读到内核缓冲区，再拷贝到用户空间，然后写入socket的内核缓冲区，最后发送到网卡。`sendfile` 直接在内核空间完成磁盘到socket的传输，跳过了用户空间的中转环节，大幅减少CPU开销和内存拷贝次数。

对于静态文件服务（图片、CSS、JS等），建议开启。对于需要修改响应内容（如SSI、压缩）的场景，sendfile不适用。

```nginx
server {
    listen 80;
    server_name static.example.com;
    root /var/www/static;

    # 静态资源启用sendfile
    sendfile on;

    location /images/ {
        sendfile on;
    }
}
```

---

### sendfile_max_chunk

**语法：** `sendfile_max_chunk size;`

**默认值：** `2m`

**适用上下文：** `http`、`server`、`location`

限制单次 `sendfile()` 调用传输的数据量。如果没有这个限制，一个大文件的传输可能在一次调用中完成，期间worker进程被完全占用，其他连接只能排队等待。

设置合理的chunk大小可以保证单个连接不会长时间独占worker，让Nginx在不同连接之间公平调度。在高并发场景下，适当减小这个值能提升整体响应速度。

```nginx
http {
    sendfile on;
    # 限制每次sendfile最多传输512KB
    sendfile_max_chunk 512k;
}
```

---

### aio

**语法：** `aio on | off | threads[=pool];`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

启用或禁用异步文件I/O。FreeBSD和Linux平台支持此功能。在Linux上，AIO需要配合 `directio` 一起使用，因为Linux的AIO仅对 `O_DIRECT` 模式的文件有效。

当 `aio` 和 `sendfile` 同时启用时，Nginx会根据文件大小自动选择策略：大于等于 `directio` 设定值的文件走AIO路径，较小的文件走sendfile路径。这种组合既保证了小文件的高速传输，又避免了sendfile处理大文件时阻塞worker的问题。

`threads` 参数启用多线程读取，将磁盘I/O操作卸载到线程池中，避免阻塞事件循环。可以指定线程池名称，不指定则使用默认池。

```nginx
http {
    # 基本AIO配置
    aio on;
    directio 8m;

    # 多线程AIO配置
    aio threads=pool_name;

    server {
        location /video/ {
            aio threads;
            directio 512k;
            sendfile on;
            # 大于512K的视频文件用AIO+线程池
            # 小于512K的缩略图用sendfile
        }
    }
}
```

---

### aio_write

**语法：** `aio_write on | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

当 `aio` 已启用时，控制是否将AIO用于写操作。目前这个功能仅在 `aio threads` 模式下有效，主要场景是写入从上游代理服务器接收到的临时文件。

启用后，Nginx把临时文件的写入操作交给线程池异步完成，主事件循环不会被磁盘写阻塞。对于反向代理缓存大量内容到磁盘的场景，这个选项能明显提升性能。

```nginx
server {
    location /proxy/ {
        proxy_pass http://backend;
        aio threads;
        aio_write on;

        # 代理响应写入临时文件时使用异步I/O
        proxy_temp_path /var/cache/nginx/temp;
    }
}
```

---

### directio

**语法：** `directio size | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

对大于等于指定大小的文件设置 `O_DIRECT` 标志，绕过操作系统页缓存直接读写磁盘。启用后自动在该location中禁用sendfile，因为两者机制不兼容。

`O_DIRECT` 在Linux上要求数据按512字节边界对齐（部分文件系统如XFS要求4K对齐）。这个选项最适合大文件服务（视频、ISO镜像等），避免大文件吃满操作系统的页缓存，把缓存空间留给更频繁访问的小文件。

```nginx
server {
    location /downloads/ {
        # 大于16MB的文件直接I/O
        directio 16m;
        aio on;

        location /downloads/video/ {
            # 视频文件设置更小的阈值
            directio 4m;
        }
    }
}
```

---

### directio_alignment

**语法：** `directio_alignment size;`

**默认值：** `512`

**适用上下文：** `http`、`server`、`location`

设置 `directio` 的内存对齐边界。默认512字节对大多数文件系统足够。在Linux上使用XFS文件系统时，需要设置为4096（4K），否则 `O_DIRECT` 操作会因对齐不符而失败。

```nginx
server {
    # XFS文件系统需要4K对齐
    directio 8m;
    directio_alignment 4096;
}
```

---

### read_ahead

**语法：** `read_ahead size;`

**默认值：** `0`

**适用上下文：** `http`、`server`、`location`

设置内核预读（read-ahead）的数据量。Nginx通过 `posix_fadvise()` （Linux）或 `fcntl(O_READAHEAD)` （FreeBSD）通知内核提前把文件内容读入页缓存。

对于顺序读取的静态文件服务场景，适当设置预读量可以减少磁盘寻道等待。默认值0表示不设置预读。在机械硬盘上效果明显，SSD上提升有限。

```nginx
server {
    location /media/ {
        read_ahead 1m;
        sendfile on;
    }
}
```

---

### output_buffers

**语法：** `output_buffers number size;`

**默认值：** `2 32k`

**适用上下文：** `http`、`server`、`location`

设置从磁盘读取响应内容时使用的缓冲区数量和单个缓冲区大小。当sendfile关闭时，Nginx需要把文件内容读入用户空间缓冲区再发送。

更大的缓冲区可以减少系统调用次数，提高读取效率，但会占用更多内存。对于大文件服务或高带宽场景，可以适当增大。

```nginx
http {
    output_buffers 4 64k;
}
```

---

### open_file_cache

**语法：** `open_file_cache off;` 或 `open_file_cache max=N [inactive=time];`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

配置文件元数据缓存，避免每次请求都执行 `stat()` 系统调用。缓存可以存储三类信息：

- 打开的文件描述符及其元数据（文件大小、修改时间等）
- 目录是否存在的信息
- 文件查找失败的错误结果

`max=N` 设定缓存条目上限，超过时用LRU淘汰最久未访问的条目。`inactive=time` 设定过期时间，在此期间未被访问的条目自动移除，默认60秒。

对于大量静态文件的站点，这个缓存能显著减少磁盘I/O和系统调用开销。

```nginx
http {
    # 最多缓存10000个条目，120秒未访问则过期
    open_file_cache max=10000 inactive=120s;

    server {
        location /static/ {
            open_file_cache max=5000 inactive=60s;
        }
    }
}
```

---

### open_file_cache_errors

**语法：** `open_file_cache_errors on | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

控制是否缓存文件查找的错误结果（文件不存在、权限拒绝等）。默认关闭是因为"文件不存在"这种错误通常不应该被缓存，除非你确定文件不会被动态创建。

开启后，频繁请求不存在的文件时不会每次都打到磁盘。适合静态资源站点，所有文件在部署时已确定存在的场景。

```nginx
http {
    open_file_cache max=2000 inactive=30s;
    open_file_cache_errors on;
}
```

---

### open_file_cache_min_uses

**语法：** `open_file_cache_min_uses number;`

**默认值：** `1`

**适用上下文：** `http`、`server`、`location`

设置缓存条目在 `inactive` 期间内至少被访问多少次才能留在缓存中。低于这个次数的条目在过期检查时被移除。

调高这个值可以过滤掉偶尔被访问的冷门文件，让缓存空间留给真正的热点文件。在文件数量多但热点集中的场景下特别有用。

```nginx
http {
    open_file_cache max=5000 inactive=60s;
    # 至少被访问3次才保留在缓存中
    open_file_cache_min_uses 3;
}
```

---

### open_file_cache_valid

**语法：** `open_file_cache_valid time;`

**默认值：** `60s`

**适用上下文：** `http`、`server`、`location`

设置缓存条目的验证间隔。每隔这段时间，Nginx会检查缓存中的文件元数据是否仍然有效（比如文件是否被修改过、是否被删除）。

间隔太短会增加 `stat()` 调用频率，间隔太长可能导致客户端拿到过时的内容。对于更新不频繁的静态站点，可以适当加长；对于经常更新的内容，保持默认或缩短。

```nginx
http {
    open_file_cache max=3000 inactive=90s;
    # 每120秒验证一次缓存有效性
    open_file_cache_valid 120s;
}
```

---

### tcp_nodelay

**语法：** `tcp_nodelay on | off;`

**默认值：** `on`

**适用上下文：** `http`、`server`、`location`

控制 `TCP_NODELAY` 选项。Nagle算法会把多个小包合并成一个大包发送以减少网络负载，但代价是增加了延迟。`TCP_NODELAY` 禁用Nagle算法，让数据立即发送。

Nginx默认在keep-alive连接上启用。对于低延迟敏感的Web应用（API接口、实时通信），保持开启。对于大批量文件传输且对延迟不敏感的场景，关闭可能略提升吞吐。

```nginx
server {
    # API服务，优先低延迟
    tcp_nodelay on;

    location /api/ {
        tcp_nodelay on;
    }
}
```

---

### tcp_nopush

**语法：** `tcp_nopush on | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

启用 `TCP_NOPUSH`（FreeBSD）或 `TCP_CORK`（Linux）选项。这个选项让数据在积累到一定量（MSS大小）或收到明确发送指令之前不发出，与 `TCP_NODELAY` 的思路正好相反。

但两者可以配合使用。`tcp_nopush` 在sendfile传输期间让数据打包成大块高效发送，传输结束后Nginx会立即关闭cork，此时 `tcp_nodelay` 生效，确保最后一个不完整的数据包也不被延迟发送。

```nginx
http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
}
```

---

### send_lowat

**语法：** `send_lowat size;`

**默认值：** `0`

**适用上下文：** `http`、`server`、`location`

如果设为非零值，Nginx尝试使用 `NOTE_LOWAT`（kqueue，FreeBSD）或 `SO_SNDLOWAT` socket选项来控制客户端socket的发送低水位。当发送缓冲区中未发送的数据量低于此值时，内核才通知Nginx可以继续写入。

实际使用中这个参数很少需要调整，保持默认值即可。在某些高并发且发送缓冲区压力大的场景下可能有调优价值。

```nginx
http {
    sendfile on;
    # 极少需要修改此参数
    send_lowat 0;
}
```

---

### 综合性能优化示例

以下配置将上述指令组合使用，兼顾小文件的高速传输和大文件的异步处理，同时利用文件缓存减少磁盘开销：

```nginx
http {
    # 文件传输优化
    sendfile on;                    # 启用sendfile
    tcp_nopush on;                  # 优化数据包发送
    tcp_nodelay on;                 # 禁用Nagle算法

    # AIO + sendfile 组合
    # 大文件(>8M)使用AIO，小文件使用sendfile
    aio on;
    directio 8m;
    sendfile_max_chunk 1m;

    # 文件缓存
    open_file_cache max=1000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;

    # 输出缓冲
    output_buffers 2 64k;

    server {
        listen 80;
        server_name example.com;
        root /var/www/html;
    }
}
```

这套配置的逻辑是：小文件通过sendfile零拷贝快速发出，大文件走AIO避免阻塞worker进程，文件元数据缓存在内存中减少磁盘stat调用，`tcp_nopush` 和 `tcp_nodelay` 配合确保数据包既高效又不延迟。根据实际文件大小分布和服务器硬件，可以调整 `directio` 阈值、缓存条目数和缓冲区大小。
# Nginx 配置指令详解（七）

## 重定向与响应控制

### absolute_redirect

**语法：** `absolute_redirect on | off;`

**默认值：** `on`

**适用上下文：** `http`、`server`、`location`

**版本：** 1.11.8+

**说明：**

控制 nginx 发出的重定向是否使用绝对 URL。启用时（默认），重定向响应中的 `Location` 头会包含完整的绝对地址（含协议、主机名和路径）。禁用后，nginx 发出的重定向将变为相对路径，只包含目标路径部分。某些反向代理场景下使用相对重定向可以避免因代理层修改主机名而导致重定向地址错误的问题。

**配置示例：**

```nginx
server {
    listen 80;
    server_name example.com;

    # 禁用绝对重定向，使用相对路径
    absolute_redirect off;

    location /old {
        return 301 /new;
    }
}
```

### port_in_redirect

**语法：** `port_in_redirect on | off;`

**默认值：** `on`

**适用上下文：** `http`、`server`、`location`

**说明：**

控制 nginx 在发出绝对重定向时是否在 URL 中包含端口号。默认情况下 nginx 会将监听端口附加到重定向 URL 中。如果 nginx 运行在非标准端口上但前端有反向代理统一处理端口，关闭此选项可以避免在重定向 URL 中暴露内部端口。此指令仅在 `absolute_redirect` 为 `on` 时有效。

**配置示例：**

```nginx
server {
    listen 8080;
    server_name example.com;

    # nginx 监听 8080，但前面有代理统一走 443
    # 禁用端口避免重定向到 example.com:8080
    port_in_redirect off;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### server_name_in_redirect

**语法：** `server_name_in_redirect on | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

**说明：**

控制 nginx 在发出绝对重定向时是否使用 `server_name` 指令中定义的主服务器名称（即第一个名称）。启用后，重定向 URL 中的主机名会被替换为 `server_name` 的主名称。禁用时（默认），nginx 使用请求中的 `Host` 头字段作为重定向的主机名。当客户端可能通过多个域名访问同一服务但希望统一重定向到规范域名时，可以启用此指令。

**配置示例：**

```nginx
server {
    listen 80;
    server_name www.example.com example.com;

    # 所有重定向统一使用 www.example.com
    server_name_in_redirect on;

    location / {
        return 301 https://www.example.com$request_uri;
    }
}
```

### chunked_transfer_encoding

**语法：** `chunked_transfer_encoding on | off;`

**默认值：** `on`

**适用上下文：** `http`、`server`、`location`

**说明：**

控制是否允许在 HTTP/1.1 响应中使用分块传输编码（chunked transfer encoding）。分块编码允许服务器在不知道响应总长度时逐块发送数据，对动态内容和流式传输非常关键。某些老旧的客户端软件或中间代理可能不支持分块编码，此时可以禁用。禁用后 nginx 会尝试缓冲完整响应再发送，或使用其他方式传输。

**配置示例：**

```nginx
server {
    listen 80;
    server_name example.com;

    # 禁用分块编码，兼容不支持 chunked 的旧客户端
    chunked_transfer_encoding off;

    location /api/ {
        proxy_pass http://backend;
    }
}
```

### etag

**语法：** `etag on | off;`

**默认值：** `on`

**适用上下文：** `http`、`server`、`location`

**版本：** 1.3.3+

**说明：**

控制是否为静态资源自动生成 `ETag` 响应头。`ETag` 是资源版本的标识符，浏览器会缓存它并在后续请求中通过 `If-None-Match` 头发送给服务器进行验证。如果 ETag 未变化，服务器返回 304 状态码，浏览器直接使用缓存，节省带宽。禁用 ETag 后浏览器将无法通过此机制验证缓存有效性。在某些 CDN 或反向代理架构中，可能需要禁用 nginx 自身的 ETag 生成以避免与上层缓存策略冲突。

**配置示例：**

```nginx
server {
    listen 80;
    server_name example.com;

    location /static/ {
        # 禁用 ETag，由 CDN 层处理缓存验证
        etag off;
        expires 30d;
    }

    location /dynamic/ {
        # 保留 ETag，支持条件请求
        etag on;
    }
}
```

### if_modified_since

**语法：** `if_modified_since off | exact | before;`

**默认值：** `exact`

**适用上下文：** `http`、`server`、`location`

**说明：**

控制 nginx 如何将响应的 `Last-Modified` 时间与请求中的 `If-Modified-Since` 头进行比较，用于条件请求处理。三种模式的行为如下：

- `off`：忽略 `If-Modified-Since` 头，始终返回完整响应，不进行 304 判断
- `exact`：精确匹配，仅当响应的修改时间等于请求头中的时间时才返回 304
- `before`：宽松匹配，当响应的修改时间小于或等于请求头中的时间时返回 304

大部分场景使用默认的 `exact` 即可。`before` 模式在文件可能被回滚到旧版本时更宽容。

**配置示例：**

```nginx
server {
    listen 80;
    server_name example.com;

    # 静态文件使用宽松匹配
    location /assets/ {
        if_modified_since before;
        expires 7d;
    }

    # API 接口忽略条件请求，始终返回完整数据
    location /api/ {
        if_modified_since off;
        proxy_pass http://backend;
    }

    # 默认精确匹配
    location /docs/ {
        if_modified_since exact;
    }
}
```

### server_tokens

**语法：** `server_tokens on | off | build | string;`

**默认值：** `on`

**适用上下文：** `http`、`server`、`location`

**说明：**

控制是否在错误页面和 `Server` 响应头中显示 nginx 版本号。启用时（默认），响应头会显示类似 `nginx/1.24.0` 的完整版本信息。这会暴露服务器软件及具体版本，给攻击者提供指纹信息。生产环境强烈建议设为 `off`，只显示 `nginx` 不显示版本号。设为 `build` 则同时显示构建信息。也可以直接指定一个自定义字符串替换默认值（需要商业版本支持）。

**配置示例：**

```nginx
http {
    # 生产环境隐藏版本号
    server_tokens off;

    server {
        listen 80;
        server_name example.com;
    }
}

# 开发环境可以保留版本信息
server {
    listen 8080;
    server_name dev.example.com;
    server_tokens on;
}
```

### msie_padding

**语法：** `msie_padding on | off;`

**默认值：** `on`

**适用上下文：** `http`、`server`、`location`

**说明：**

控制是否为 MSIE（Microsoft Internet Explorer）客户端在状态码大于 400 的错误响应中添加 HTML 注释，将响应内容填充到 512 字节。这是为了解决旧版 IE 浏览器的一个特性：当错误响应体小于 512 字节时，IE 会用自带的"友好错误页面"替换服务器返回的内容，导致自定义错误页无法正常展示。现代浏览器已无此问题，如果不需要兼容旧版 IE 可以关闭以节省少量带宽。

**配置示例：**

```nginx
server {
    listen 80;
    server_name example.com;

    # 现代站点无需兼容旧 IE，关闭填充
    msie_padding off;

    error_page 404 /404.html;
    location = /404.html {
        internal;
    }
}
```

### msie_refresh

**语法：** `msie_refresh on | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

**说明：**

控制是否为 MSIE 客户端使用 meta refresh 标签代替 HTTP 重定向。启用后，当 nginx 需要对 MSIE 客户端进行重定向时，会返回包含 `<meta http-equiv="Refresh" ...>` 标签的 HTML 页面，而不是标准的 301/302 重定向。这同样是为了兼容旧版 IE 的特殊行为。现代场景中一般不需要启用。

**配置示例：**

```nginx
server {
    listen 80;
    server_name example.com;

    # 兼容旧版 IE 的重定向方式
    msie_refresh on;

    location /old-path {
        return 301 /new-path;
    }
}
```

### postpone_output

**语法：** `postpone_output size;`

**默认值：** `1460`

**适用上下文：** `http`、`server`、`location`

**说明：**

设置 nginx 推迟向客户端发送数据的最小字节数。当此值大于 0 时，nginx 会尽可能将小数据块缓冲合并，直到累积到指定大小再一次性发送。默认值 1460 字节正好是以太网 MTU（1500）减去 TCP 头（40）的典型值，可以在一次 TCP 包中发送完整数据，减少小包传输的开销。设为 0 可以完全禁用推迟行为，数据产生后立即发送，适合对延迟极度敏感的场景，但会增加 TCP 包的数量。

**配置示例：**

```nginx
http {
    # 默认值，适合大多数场景
    postpone_output 1460;

    server {
        listen 80;
        server_name example.com;

        # 实时流式接口，禁用推迟以降低延迟
        location /stream/ {
            postpone_output 0;
            proxy_pass http://backend;
        }
    }
}
```

---

## 路径与安全

### disable_symlinks

**语法：** `disable_symlinks off;` 或 `disable_symlinks on | if_not_owner [from=part];`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

**说明：**

控制 nginx 在打开文件时如何处理路径中的符号链接。符号链接可能被利用来访问 web 根目录之外的文件，在共享主机或多租户环境中尤其危险。各选项含义：

- `off`：不做任何检查，允许所有符号链接（默认）
- `on`：如果路径中任何组件是符号链接，拒绝访问并返回 403
- `if_not_owner`：仅当符号链接及其指向目标的所有者不同时拒绝访问
- `from=part`：仅在路径中匹配指定前缀之后的部分检查符号链接，前缀部分允许符号链接

注意：启用此功能会增加每次文件访问时的额外系统调用，可能影响性能。

**配置示例：**

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www;

    # 严格禁止所有符号链接
    location /secure/ {
        disable_symlinks on;
    }

    # 仅在所有者不匹配时拒绝
    location /shared/ {
        disable_symlinks if_not_owner;
    }

    # 允许 /var/www 内的符号链接，之后的路径禁止
    location /downloads/ {
        disable_symlinks on from=/var/www;
    }
}
```

### merge_slashes

**语法：** `merge_slashes on | off;`

**默认值：** `on`

**适用上下文：** `http`、`server`

**说明：**

控制是否将请求 URI 中连续的两个或多个斜杠压缩为单个斜杠。默认启用，URI `//foo///bar` 会被规范化为 `/foo/bar`。这个行为有助于防止路径遍历攻击和路由混乱。但如果后端应用使用双斜杠表达特殊语义（如某些 REST API），则需要关闭此选项以保留原始 URI。此指令必须谨慎关闭，因为禁用后可能导致通过注入多余斜杠绕过某些安全规则。

**配置示例：**

```nginx
http {
    # 默认合并斜杠
    merge_slashes on;

    server {
        listen 80;
        server_name example.com;

        # 后端 API 依赖双斜杠语义
        location /api/ {
            merge_slashes off;
            proxy_pass http://backend;
        }
    }
}
```

### recursive_error_pages

**语法：** `recursive_error_pages on | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

**说明：**

控制是否允许 `error_page` 指令进行多次重定向。默认情况下，nginx 只允许一次 error_page 重定向，如果在错误页面处理过程中又发生了错误，nginx 会直接返回原始错误码。启用后，nginx 允许在 error_page 重定向目标上再次触发 error_page 处理。但重定向次数有上限，超过后 nginx 返回 500 错误。适用于需要将多种错误统一汇聚到一个处理页面的场景。

**配置示例：**

```nginx
server {
    listen 80;
    server_name example.com;

    # 允许多次 error_page 重定向
    recursive_error_pages on;

    # 先按状态码分发到不同错误页面
    error_page 403 /errors/403.html;
    error_page 404 /errors/404.html;

    # 如果错误页面本身也找不到，统一跳转到通用错误页
    location /errors/ {
        internal;
        error_page 404 /fallback.html;
    }

    location = /fallback.html {
        internal;
        root /var/www/errors;
    }
}
```

### satisfy

**语法：** `satisfy all | any;`

**默认值：** `all`

**适用上下文：** `http`、`server`、`location`

**说明：**

当同一个 location 同时配置了多种访问控制方式时（如 IP 白名单 `allow/deny`、HTTP 基本认证 `auth_basic`、子请求认证 `auth_request` 等），此指令决定各控制方式之间的逻辑关系：

- `all`：所有访问控制方式都必须通过，相当于逻辑"与"
- `any`：任意一种访问控制方式通过即可，相当于逻辑"或"

典型用法：IP 白名单内的用户无需密码直接访问，其他 IP 的用户需要密码验证。

**配置示例：**

```nginx
server {
    listen 80;
    server_name example.com;

    # IP 白名单内的直接放行，其他 IP 需要密码
    location /admin/ {
        satisfy any;

        # 访问控制方式一：IP 白名单
        allow 192.168.1.0/24;
        allow 10.0.0.0/8;
        deny all;

        # 访问控制方式二：HTTP 基本认证
        auth_basic "Admin Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```

### auth_delay

**语法：** `auth_delay time;`

**默认值：** `0s`

**适用上下文：** `http`、`server`、`location`

**版本：** 1.17.10+

**说明：**

设置对返回 401 状态码的未授权请求的延迟处理时间。这是一种安全措施，用于缓解计时攻击（timing attack）。攻击者可能通过观察服务器响应时间的微小差异来推断认证相关的信息。设置延迟后，nginx 会在发送 401 响应之前等待指定的时间，使所有认证失败的响应时间趋于一致。当访问受 `auth_basic`、`auth_request` 或 `auth_jwt` 限制时此指令生效。

**配置示例：**

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    # 对认证失败的请求统一延迟 1 秒响应
    auth_delay 1s;

    location /private/ {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }

    location /api/ {
        auth_request /auth;
        proxy_pass http://backend;
    }

    location = /auth {
        internal;
        proxy_pass http://auth-service/validate;
    }
}
```

---

## DNS 解析

### resolver

**语法：** `resolver address ... [valid=time] [ipv4=on|off] [ipv6=on|off] [status_zone=zone];`

**默认值：** —

**适用上下文：** `http`、`server`、`location`

**说明：**

配置 nginx 用于将上游服务器域名解析为 IP 地址的 DNS 服务器。当 `proxy_pass` 等指令使用域名而非 IP 时，nginx 需要通过 DNS 解析获取实际地址。可指定多个 DNS 地址，nginx 会轮询使用。各参数说明：

- `valid=time`：设置 DNS 解析结果的缓存时间，覆盖 DNS 记录自身的 TTL
- `ipv4=on|off`：控制是否查询 A 记录（IPv4 地址）
- `ipv6=on|off`：控制是否查询 AAAA 记录（IPv6 地址）
- `status_zone=zone`：启用 DNS 解析统计（需商业版本）

如果不配置 resolver，当 proxy_pass 中使用变量构建域名时 nginx 将无法启动或解析失败。

**配置示例：**

```nginx
http {
    # 使用本地 DNS 缓存服务
    resolver 127.0.0.1 valid=30s ipv6=off;

    server {
        listen 80;
        server_name example.com;

        # 动态解析后端地址
        location /api/ {
            set $backend api-backend.internal;
            proxy_pass http://$backend;
        }
    }
}

server {
    listen 80;

    # 使用公共 DNS，缓存 60 秒
    location /proxy/ {
        resolver 8.8.8.8 8.8.4.4 valid=60s;
        set $target $arg_target;
        proxy_pass http://$target;
    }
}
```

### resolver_timeout

**语法：** `resolver_timeout time;`

**默认值：** `30s`

**适用上下文：** `http`、`server`、`location`

**说明：**

设置 DNS 解析的超时时间。如果 resolver 指定的 DNS 服务器在超时时间内未返回解析结果，nginx 将认为解析失败并返回 502（Bad Gateway）错误给客户端。在网络延迟较高或 DNS 服务器响应较慢的环境中，可能需要适当增大此值。反之，在响应要求快速失败的场景中可以减小超时时间。

**配置示例：**

```nginx
http {
    resolver 8.8.8.8 8.8.4.4;
    # 缩短 DNS 超时以快速失败
    resolver_timeout 5s;

    server {
        listen 80;
        server_name example.com;

        location /api/ {
            set $backend $arg_service.internal;
            proxy_pass http://$backend;
        }
    }
}
```

---

## 性能调优

### connection_pool_size

**语法：** `connection_pool_size size;`

**默认值：** 256（32 位平台）/ 512（64 位平台）

**适用上下文：** `http`、`server`

**说明：**

设置每个连接的内存池大小。nginx 为每个客户端连接在创建时分配一块固定大小的内存池，用于存放连接相关的数据结构。增大此值对性能影响极小，一般不应调整。在连接数极大（数万级别）且内存紧张的场景中，可以适当减小以节省内存，但过小可能导致内存池不够用，触发更频繁的内存分配。

**配置示例：**

```nginx
http {
    # 默认值即可，一般不需要修改
    connection_pool_size 512;

    server {
        listen 80;
        server_name example.com;
    }
}
```

### request_pool_size

**语法：** `request_pool_size size;`

**默认值：** `4k`

**适用上下文：** `http`、`server`

**说明：**

设置每个请求的内存池大小。nginx 为每个 HTTP 请求分配独立的内存池，用于存储请求处理过程中的临时数据。与 connection_pool_size 类似，此参数对整体性能影响极小，官方文档明确建议一般不应使用此指令进行调优。只有在极端优化的场景中才有必要关注。

**配置示例：**

```nginx
http {
    # 默认值即可
    request_pool_size 4k;

    server {
        listen 80;
        server_name example.com;
    }
}
```

### variables_hash_bucket_size

**语法：** `variables_hash_bucket_size size;`

**默认值：** 64

**适用上下文：** `http`

**说明：**

设置变量哈希表的桶大小。nginx 使用哈希表存储所有可用的变量名。哈希表的桶大小应与处理器缓存行大小匹配，通常为 64 字节。如果配置中使用了大量自定义变量（通过 `set`、`map` 等指令），导致变量名哈希冲突增加，可能需要增大此值。调整时建议以处理器缓存行大小的倍数递增。

**配置示例：**

```nginx
http {
    # 使用大量自定义变量时可能需要增大
    variables_hash_bucket_size 128;
    variables_hash_max_size 2048;

    map $uri $category {
        ~^/news/     news;
        ~^/blog/     blog;
        ~^/docs/     docs;
        default      other;
    }

    server {
        listen 80;
        server_name example.com;
    }
}
```

### variables_hash_max_size

**语法：** `variables_hash_max_size size;`

**默认值：** `1024`

**适用上下文：** `http`

**说明：**

设置变量哈希表的最大大小。当配置中定义了大量变量时，需要增大此值以容纳所有哈希条目。如果哈希表空间不足，nginx 启动时会输出警告信息提示增大此值。

**配置示例：**

```nginx

http {
    variables_hash_max_size 2048;
    variables_hash_bucket_size 128;
}
```

### server_names_hash_bucket_size

**语法：** `server_names_hash_bucket_size size;`

**默认值：** 32（32 位平台）/ 64（64 位平台）

**适用上下文：** `http`

**说明：**

设置服务器名称哈希表的桶大小。nginx 使用哈希表来快速匹配请求的 `Host` 头与对应的 `server_name`。桶大小取决于 CPU 缓存行大小。当 `server_name` 中包含很长的域名时（如超过 32 字节），需要增大此值。如果配置了带长域名的虚拟主机后 nginx 启动失败并提示 "could not build the server_names_hash"，就是此值不够。

**配置示例：**

```nginx
http {
    # 域名较长时增大桶大小
    server_names_hash_bucket_size 128;

    server {
        listen 80;
        server_name this-is-a-very-long-subdomain-name.example.com;
    }
}
```

### server_names_hash_max_size

**语法：** `server_names_hash_max_size size;`

**默认值：** `512`

**适用上下文：** `http`

**说明：**

设置服务器名称哈希表的最大大小。当配置了大量虚拟主机（数百甚至数千个 `server_name`）时，需要增大此值。nginx 启动时如果发现哈希表容量不足会报错提示。

**配置示例：**

```nginx
http {
    # 托管大量虚拟主机
    server_names_hash_max_size 4096;
    server_names_hash_bucket_size 64;
}
```

### types_hash_bucket_size

**语法：** `types_hash_bucket_size size;`

**默认值：** 64

**适用上下文：** `http`、`server`、`location`

**说明：**

设置 MIME 类型哈希表的桶大小。nginx 使用哈希表将文件扩展名映射到 MIME 类型（如 `.html` 对应 `text/html`）。桶大小通常应与处理器缓存行大小一致。当添加了大量自定义 MIME 类型时，可能需要增大此值。

**配置示例：**

```nginx
http {
    types_hash_bucket_size 64;
    types_hash_max_size 1024;

    # 自定义 MIME 类型
    types {
        application/wasm  wasm;
        application/json  map;
    }
}
```

### types_hash_max_size

**语法：** `types_hash_max_size size;`

**默认值：** `1024`

**适用上下文：** `http`、`server`、`location`

**说明：**

设置 MIME 类型哈希表的最大大小。当自定义了大量文件扩展名到 MIME 类型的映射时可能需要增大。

**配置示例：**

```nginx
http {
    types_hash_max_size 2048;
}
```

---

## 日志与调试

### log_not_found

**语法：** `log_not_found on | off;`

**默认值：** `on`

**适用上下文：** `http`、`server`、`location`

**说明：**

控制是否在 `error_log` 中记录文件未找到（404）错误。默认启用，每次请求的文件不存在都会记录一条日志。某些场景中这会产生大量无意义的日志条目，例如客户端频繁请求 `favicon.ico`、`robots.txt` 或各种探测路径。可以针对这些位置关闭日志以减少日志噪音。

**配置示例：**

```nginx
server {
    listen 80;
    server_name example.com;

    # 常见探测文件不记录 404
    location = /favicon.ico {
        log_not_found off;
        return 204;
    }

    location = /robots.txt {
        log_not_found off;
        return 204;
    }

    # 正常请求保留 404 日志
    location / {
        try_files $uri $uri/ =404;
    }
}
```

### log_subrequest

**语法：** `log_subrequest on | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`、`location`

**说明：**

控制是否在 `access_log` 中记录子请求的访问日志。子请求是由 `SSI`（Server Side Includes）、`auth_request`、`add_before_body`/`add_after_body` 等功能触发的内部请求。默认不记录，避免主请求的日志被大量子请求日志淹没。调试 SSI 或认证子请求的行为时可以临时启用。

**配置示例：**

```nginx
server {
    listen 80;
    server_name example.com;

    # 调试 auth_request 子请求
    location /protected/ {
        log_subrequest on;
        auth_request /auth;
        proxy_pass http://backend;
    }

    location = /auth {
        internal;
        proxy_pass http://auth-service/validate;
    }
}
```

### ignore_invalid_headers

**语法：** `ignore_invalid_headers on | off;`

**默认值：** `on`

**适用上下文：** `http`、`server`

**说明：**

控制是否忽略包含无效名称的请求头字段。有效的请求头名称应由英文字母、数字和连字符组成。启用时（默认），nginx 会静默丢弃无效的请求头。禁用时，如果请求中包含无效头名称，nginx 将返回 400（Bad Request）错误并拒绝请求。如果后端应用需要接收非标准格式的请求头，可以启用此指令让 nginx 放行。此指令在 `http` 和 `server` 级别有效，不能用于 `location` 块。

**配置示例：**

```nginx
http {
    # 默认忽略无效头
    ignore_invalid_headers on;

    server {
        listen 80;
        server_name example.com;
    }

    # 需要严格校验的服务
    server {
        listen 80;
        server_name strict.example.com;
        ignore_invalid_headers off;
    }
}
```

### underscores_in_headers

**语法：** `underscores_in_headers on | off;`

**默认值：** `off`

**适用上下文：** `http`、`server`

**说明：**

控制是否允许客户端请求头字段名称中包含下划线。HTTP 规范并没有禁止下划线，但实际中建议使用连字符代替。默认禁用时，带下划线的请求头会被视为无效字段而被丢弃（当 `ignore_invalid_headers` 为 `on` 时）。许多应用框架（如 Django）习惯使用 `X_Forwarded_For` 这样的下划线头，此时需要启用此指令。此指令在 `http` 和 `server` 级别有效。

**配置示例：**

```nginx
http {
    # 允许请求头中使用下划线
    underscores_in_headers on;

    server {
        listen 80;
        server_name example.com;

        location /api/ {
            # 后端依赖 X_Custom_Header 之类的头
            proxy_pass http://backend;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

---

## 子请求与其他

### subrequest_output_buffer_size

**语法：** `subrequest_output_buffer_size size;`

**默认值：** `4k`

**适用上下文：** `http`、`server`、`location`

**说明：**

设置用于存储子请求响应体的缓冲区大小。当子请求（如 SSI include、`auth_request` 等）的响应体需要被读取到内存时，使用此缓冲区存储。如果子请求的响应体大小超过此值，响应数据会被写入临时文件。对于已知的子请求响应较小（如认证返回的简单 JSON），保持默认即可。如果子请求返回的内容较大但又不想落盘，可以适当增大。

**配置示例：**

```nginx
http {
    subrequest_output_buffer_size 8k;

    server {
        listen 80;
        server_name example.com;

        # auth_request 子请求返回较大的 JSON
        location /api/ {
            subrequest_output_buffer_size 16k;
            auth_request /auth;
            proxy_pass http://backend;
        }

        location = /auth {
            internal;
            proxy_pass http://auth-service/validate;
        }
    }
}
```

### early_hints

**语法：** `early_hints string ...;`

**默认值：** —

**适用上下文：** `http`、`server`、`location`

**版本：** 1.29.0+

**说明：**

定义将 103（Early Hints）响应传递给客户端的条件。Early Hints 是 HTTP/2 和 HTTP/3 的一个特性，允许服务器在发送最终响应之前提前向客户端推送资源提示（如 preload 链接），让浏览器提前开始加载关键资源，减少页面渲染等待时间。如果至少一个值不为空且不为 `"0"`，nginx 就会将 103 响应传递给客户端。这通常与 `add_header` 配合使用来设置 `Link` 头。

**配置示例：**

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # 当 Link 头存在时发送 Early Hints
    early_hints $link_header;

    location / {
        # 设置 preload 提示
        set $link_header "</css/main.css>; rel=preload; as=style";
        add_header Link $link_header;

        # 同时添加多个资源提示
        # set $link_header "</css/main.css>; rel=preload; as=style, </js/app.js>; rel=preload; as=script";

        proxy_pass http://backend;
    }

    # 不需要 Early Hints 的静态资源
    location /static/ {
        early_hints "";
        expires 30d;
    }
}
```
# Nginx 配置指南（八）：内嵌变量、配置示例与最佳实践

## 内嵌变量参考

ngx_http_core_module 模块支持内嵌变量，名称与 Apache Server 变量匹配。主要包括表示客户端请求头字段的变量（如 `$http_user_agent`、`$http_cookie` 等），以及其他变量。

下面按功能分组列出所有内嵌变量。

### 请求相关变量

| 变量 | 说明 |
|---|---|
| `$arg_`name | 请求行中的 name 参数 |
| `$args` | 请求行中的参数 |
| `$query_string` | 同 `$args` |
| `$is_args` | 如果请求行有参数则为 `?`，否则为空字符串 |
| `$request` | 完整的原始请求行 |
| `$request_body` | 请求体。在 proxy_pass、fastcgi_pass 等处理时可用 |
| `$request_body_file` | 存储请求体的临时文件名 |
| `$request_completion` | 请求完成则为 `OK`，否则为空字符串 |
| `$request_filename` | 基于 root/alias 和请求 URI 的文件路径 |
| `$request_id` | 16 个随机字节生成的唯一请求标识符（十六进制） |
| `$request_length` | 请求长度（包括请求行、头部和请求体） |
| `$request_method` | 请求方法，通常为 `GET` 或 `POST` |
| `$request_time` | 请求处理时间（秒，毫秒精度） |
| `$request_uri` | 完整的原始请求 URI（含参数） |
| `$uri` | 当前请求中的 URI（规范化后）。在内部重定向或使用 index 时可能变化 |
| `$document_uri` | 同 `$uri` |
| `$scheme` | 请求协议，`http` 或 `https` |
| `$limit_rate` | 设置此变量可启用响应速率限制 |
| `$request_port` | URI 权限组件中的端口号或 Host 头中的端口号（v1.29.3） |
| `$is_request_port` | 如果 `$request_port` 非空则为 `:`，否则为空字符串（v1.29.3） |

### 客户端与连接变量

| 变量 | 说明 |
|---|---|
| `$binary_remote_addr` | 客户端地址的二进制形式，IPv4 为 4 字节，IPv6 为 16 字节 |
| `$body_bytes_sent` | 发送给客户端的字节数（不含响应头） |
| `$bytes_sent` | 发送给客户端的字节数 |
| `$connection` | 连接序列号 |
| `$connection_requests` | 通过一个连接发出的当前请求数 |
| `$connection_time` | 连接时间（秒，毫秒精度）（v1.19.10） |
| `$pipe` | 如果请求是流水线化的则为 `p`，否则为 `.` |
| `$remote_addr` | 客户端地址 |
| `$remote_port` | 客户端端口 |
| `$remote_user` | Basic 认证提供的用户名 |
| `$status` | 响应状态码 |

### 服务器变量

| 变量 | 说明 |
|---|---|
| `$document_root` | 当前请求的 root 或 alias 指令的值 |
| `$realpath_root` | root 或 alias 对应绝对路径（解析所有符号链接） |
| `$server_addr` | 接受请求的服务器地址。通常需要系统调用 |
| `$server_name` | 接受请求的服务器名称 |
| `$server_port` | 接受请求的服务器端口 |
| `$server_protocol` | 请求协议，通常为 `HTTP/1.0`、`HTTP/1.1`、`HTTP/2.0` 或 `HTTP/3.0` |

### HTTP 头部变量

| 变量 | 说明 |
|---|---|
| `$content_length` | `Content-Length` 请求头字段 |
| `$content_type` | `Content-Type` 请求头字段 |
| `$cookie_`name | 名称对应的 cookie 值 |
| `$host` | 优先级：请求行中的主机名 > Host 请求头 > 匹配请求的服务器名称 |
| `$hostname` | 主机名 |
| `$http_`name | 任意请求头字段（名称小写，破折号替换为下划线） |
| `$https` | SSL 模式下为 `on`，否则为空字符串 |
| `$sent_http_`name | 任意响应头字段 |
| `$sent_trailer_`name | 响应末尾发送的任意字段（v1.13.2） |

### PROXY 协议变量

| 变量 | 说明 |
|---|---|
| `$proxy_protocol_addr` | PROXY 协议头中的客户端地址 |
| `$proxy_protocol_port` | PROXY 协议头中的客户端端口 |
| `$proxy_protocol_server_addr` | PROXY 协议头中的服务器地址 |
| `$proxy_protocol_server_port` | PROXY 协议头中的服务器端口 |
| `$proxy_protocol_tlv_`name | PROXY 协议头中的 TLV（v1.23.2） |

### 时间与系统变量

| 变量 | 说明 |
|---|---|
| `$msec` | 当前时间（秒，毫秒精度） |
| `$nginx_version` | nginx 版本号 |
| `$pid` | worker 进程的 PID |
| `$time_iso8601` | 本地时间 ISO 8601 格式 |
| `$time_local` | 本地时间 Common Log Format |

### TCP 信息变量

| 变量 | 说明 |
|---|---|
| `$tcpinfo_rtt` | 客户端 TCP 连接的 RTT |
| `$tcpinfo_rttvar` | RTT 方差 |
| `$tcpinfo_snd_cwnd` | 发送拥塞窗口 |
| `$tcpinfo_rcv_space` | 接收缓冲区空间 |

---

## 典型配置示例

### 1. 静态文件服务器

```nginx
server {
    listen 80;
    server_name static.example.com;
    root /var/www/static;

    # 文件传输优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    # 文件缓存
    open_file_cache max=2000 inactive=20s;
    open_file_cache_valid 60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;

    # 静态资源缓存策略
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # 禁用日志
    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    # Gzip 压缩
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 1024;
}
```

### 2. 反向代理基础配置

```nginx
upstream backend {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}

server {
    listen 80;
    server_name app.example.com;

    # 客户端请求限制
    client_max_body_size 50m;
    client_body_buffer_size 128k;
    client_body_timeout 30s;

    # 代理配置
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # API 限速
    location /api/ {
        limit_rate_after 500k;
        limit_rate 50k;
        proxy_pass http://backend;
    }
}
```

### 3. HTTPS 服务器

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name secure.example.com;

    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # 安全配置
    server_tokens off;

    root /var/www/secure;

    location / {
        try_files $uri $uri/ /index.html;
    }
}

# HTTP → HTTPS 重定向
server {
    listen 80;
    server_name secure.example.com;
    return 301 https://$host$request_uri;
}
```

### 4. 性能优化配置

```nginx
http {
    # 基础优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    # 连接优化
    keepalive_timeout 65s 60s;
    keepalive_requests 2000;
    keepalive_time 2h;
    reset_timedout_connection on;

    # AIO 大文件处理
    aio on;
    directio 8m;
    directio_alignment 4k;
    output_buffers 2 64k;

    # 文件缓存
    open_file_cache max=5000 inactive=30s;
    open_file_cache_valid 60s;
    open_file_cache_min_uses 3;
    open_file_cache_errors on;

    # 请求处理
    client_body_buffer_size 16k;
    client_header_buffer_size 2k;
    large_client_header_buffers 4 16k;

    server {
        listen 80;
        server_name optimized.example.com;
        root /var/www/html;
    }
}
```

### 5. 安全加固配置

```nginx
server {
    listen 80;
    server_name secure.example.com;

    # 隐藏版本号
    server_tokens off;

    # 安全头部
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # 请求大小限制
    client_max_body_size 10m;
    client_body_buffer_size 16k;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;

    # 超时设置
    client_body_timeout 12s;
    client_header_timeout 12s;
    send_timeout 10s;

    # 连接管理
    keepalive_timeout 30s;
    keepalive_requests 500;
    reset_timedout_connection on;

    # 禁用不必要的 HTTP 方法
    limit_except GET HEAD POST {
        deny all;
    }

    # 符号链接安全
    disable_symlinks on;

    # 允许下划线请求头
    underscores_in_headers on;

    root /var/www/html;

    location / {
        try_files $uri $uri/ =404;
    }

    # 错误页面
    error_page 403 404 /404.html;
    error_page 500 502 503 504 /50x.html;

    location = /404.html {
        internal;
    }

    location = /50x.html {
        internal;
    }
}
```

---

## 最佳实践总结

1. **启用 sendfile 和 tcp_nopush** — 利用内核级文件传输，减少用户空间和内核空间之间的数据拷贝。

2. **合理设置 keepalive** — 根据业务场景调整 `keepalive_timeout` 和 `keepalive_requests`，平衡连接复用和资源占用。

3. **隐藏版本号** — 生产环境务必设置 `server_tokens off`，避免暴露 nginx 版本信息。

4. **限制请求体大小** — 使用 `client_max_body_size` 防止大文件上传耗尽服务器资源。

5. **配置文件缓存** — 使用 `open_file_cache` 系列指令缓存频繁访问的文件描述符，减少磁盘 I/O。

6. **使用 try_files 优雅处理请求** — 优先检查文件是否存在，再决定转发到后端，减少不必要的后端请求。

7. **精确匹配优先** — 善用 `location =` 精确匹配高频路径（如首页），避免不必要的正则匹配开销。

8. **超时时间不宜过长** — `client_body_timeout`、`client_header_timeout`、`send_timeout` 应根据业务合理设置，避免连接长时间占用。

9. **AIO 与 sendfile 组合使用** — 大文件使用 AIO + directio，小文件使用 sendfile，兼顾性能与内存使用。

10. **日志分级控制** — 对于高频但不重要的请求（如 favicon.ico），使用 `log_not_found off` 减少日志写入。
