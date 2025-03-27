---
title: Linux curl 基础用法
date: 2024-08-14 20:26:42
tags: linux
categories: Posts
---
# Curl 基础用法

curl 是一个很全面的命令行浏览器，功能丰富

## 基本用法
- -f/--fail  在 HTTP 错误时返回非零状态码， echo $?, 不用-f即使HTTP错误状态码也为0
- -s   静默输出
- -S   输出错误消息，配合s使用，-sS，静默输出状态下也输出错误消息

    -sSf 返回错误状态码 如：curl: (22) The requested URL returned error: 400

	不加S不会显示错误输出
    
- -k  忽略SSL证书验证
- --connect-timeout: 设置连接超时的秒数。这是指从开始请求到完成 TCP 连接的时间限制。如果在指定时间内无法建立连接，curl 将会中止。
- --max-time: 设置整个请求的最大时间（包括连接时间、传输时间等）。如果超过了这个时间，curl 将中止请求。
- -H 添加header
- -X GET/POST  指定请求方法
- -u username:passwd  指定账号密码
- -d 发送POST请求时附带的数据
- -v 详细模式发送请求
- -o 将下载的文件保存为指定文件名
- -O 保存为原始文件名
- -F 上传文件  header使用 Content-Type multipart/form-data
- -C 断点续传
- -I 显示响应码及源码
- -i 只显示响应码
- -c 记录cookie或session
- -b 请求时使用携带的cookie

## 使用示例

```bash
# post请求携带参数
curl -X POST -d "key=value" http://127.0.0.1

# json 格式请求
curl -X POST -d '{"key": "value"}' -H "Content-Type: application/json" http://127.0.0.1

# 访问 https时忽略证书 
curl -sSfL -k https://127.0.0.1

# 记录返回的cookie
curl -c cookies.txt -X POST -d '{"key": "value"}' -H "Content-Type: application/json" http://127.0.0.1

# 请求时携带文件中的cookie值
curl -b cookies.txt -X POST -d '{"key": "value"}' -H "Content-Type: application/json" http://127.0.0.1

# 表单上传文件
curl -X POST -F "files=@/path/to/local/file" http://127.0.0.1/upload-files

# 如果请求返回是一个脚本，直接执行
curl -s http://127.0.0.1 -O- | bash
```