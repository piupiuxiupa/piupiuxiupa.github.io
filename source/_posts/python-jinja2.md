---
title: Python jinja2
date: 2025-03-05 14:10:03
tags: python
categories: Posts
---

# jinjia2 模板引擎

jinjia2 可用于生成 html、xml、yaml 等格式的文件

## 基本用法

```python
from jinja2 import Template
template = Template('Hello {{ name }}!')
template.render(name='John Doe')
```

## 基本结构

模板包含`变量`或`表达式` ，这两者在模板求值的时候会被替换为值。模板中还有标签，控制模板的逻辑。模板语法的大量灵感来自于 Django 和 Python 。

两类分隔符：
- `{{  }}`
- `{%  %}`

> 在前或后加上-可以用于去掉空格或空行，如 `{{- -}}` `{{%-  -%}}`

## 过滤器

- safe 渲染时值不转义

- capitialize 把值的首字母转换成大写，其他字母转换为小写

- lower 把值转换成小写形式

- upper 把值转换成大写形式

- title 把值中每个单词的首字母都转换成大写

- trim 把值的首尾空格去掉

- striptags 渲染之前把值中所有的HTML标签都删掉

- join 拼接多个值为字符串

- replace 替换字符串的值

- round 默认对数字进行四舍五入，也可以用参数进行控制

- int 把值转换成整型

## 模板继承

### 基础模板

```python
<html lang="en">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    {% block head %}
    <link rel="stylesheet" href="style.css" />
    <title>{% block title %}{% endblock %} - My Webpage</title>
    {% endblock %}
</head>
<body>
    <div id="content">{% block content %}{% endblock %}</div>
    <div id="footer">
        {% block footer %}
        &copy; Copyright 2008 by <a href="http://domain.invalid/">you</a>.
        {% endblock %}
    </div>
</body>
```

### 子模板

```python
{% extends "base.html" %}
{% block title %}Index{% endblock %}
{% block head %}
    {{ super() }}
    <style type="text/css">
        .important { color: #336699; }
    </style>
{% endblock %}
{% block content %}
    <h1>Index</h1>
    <p class="important">
      Welcome on my awesome homepage.
    </p>
{% endblock %}
```

## 控制结构

### for 循环

在一个 for 循环块中你可以访问这些特殊的变量:

- loop.index 当前循环迭代的次数（从 1 开始）

- loop.index0 当前循环迭代的次数（从 0 开始）

- loop.revindex 到循环结束需要迭代的次数（从 1 开始）

- loop.revindex0 到循环结束需要迭代的次数（从 0 开始）

- loop.first 如果是第一次迭代，为 True 。

- loop.last 如果是最后一次迭代，为 True 。

- loop.length 序列中的项目数。

- loop.cycle 在一串序列间期取值的辅助函数。

```python
{% for item in list %}
    {{ item }}
{% endfor %}
```

### if 条件判断

- `==` 比较两个对象是否相等。

- `!=` 比较两个对象的不等式。

- `>` 如果左侧大于右侧，则为true。

- `>=` 如果左侧大于或等于右侧，则为true。

- `<` 如果左侧小于右侧，则为true。

- `<=` 如果左侧小于或等于右侧，则为true

- and 如果左右操作数为真，则返回true。

- or 如果左操作数或右操作数为真，则返回true。

- not 非，not x ：如果 x 为 True，返回 False 。如果 x 为 False，它返回 True。

```python
{% if path != "" and path is not none %}
{% endif %}
```

## 宏

可以看做 jinjia2 中的函数，用于取代重复的工作