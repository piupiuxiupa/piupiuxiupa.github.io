---
title: jq 常用操作介绍
date: 2024-12-06 11:04:35
tags: tools
categories: Posts
---

# jq 命令基本用法

jq 是一个轻量级且功能强大的命令行 JSON 处理工具，允许用户灵活地解析、过滤、格式化和转换 JSON 数据。

随着微服务架构的兴起以及 JSON 数据格式的广泛使用，jq 命令的使用频率也在变高。

> https://github.com/Kas-tle/java2bedrock.sh/blob/main/converter.sh
>
> 用到了大量 jq 命令，可以学习借鉴。
>
> Top 10 open source and public projects attracting the most first-time contributors in 2024 on GitHub

## 基本语法

jq [选项] <过滤表达式> [文件名]

## 常用操作

1. 格式化输出

    ```bash
    # 格式化 JSON 数据为可读格式 
    jq '.' data.json
    ```

2. 提取字段值

    ```bash
    # 提取 name字段值
    jq '.name' data.json
    # 提取多个字段值
    jq '.name, .age' data.json
    jq '.[] | {name: .name, city: .city}' data.json
    ```

3. 提取嵌套字段

    ```bash
    # 提取 skills 数组的第一个值
    jq '.skills[0]' data.json
    ```

4. 过滤数组

    ```bash
    # 输出数组中的所有值。
    jq '.skills[]' data.json
    ```

## 复杂操作

1. 条件筛选

    ```bash
    # 输出 age 大于 25 的数据
    jq 'select(.age > 25)' data.json
    ```

2. 计算字段

    ```bash
    # 计算 age 字段的两倍
    jq '.age * 2' data.json
    ```

3. 修改 JSON 数据

    ```bash
    # 将 age 字段的值修改为 35
    jq '.age = 35' data.json
    ```

4. 添加新字段

    ```bash
    # 给 JSON 数据添加 city 字段
    jq '. + {city: "New York"}' data.json
    ```

5. 结合管道命令

   - 从命令行获取数据并处理：

    ```bash
    # 作用：从 API 获取数据并筛选价格大于 100 的项目
    curl -s https://api.example.com/data | jq '.items[] | select(.price > 100)'
    ```

## 常用选项

- -r：去掉输出中的引号

    ```bash
    jq -r '.name' data.json
    ```

- -c：压缩格式输出

    ```bash
    jq -c '.' data.json
    ```
