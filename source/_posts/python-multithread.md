---
title: Python 多线程编程
date: 2025-01-03 15:15:32
tags: python
categories: Posts
---

# Python 多线程编程

## 线程分配公式

`concurrent.futures.ThreadPoolExecutor` 默认线程数 `min(32, (os.cpu_count() or 1) + 4)`

- 计算密集型
  - 线程数 ≈ CPU 核心数，避免过多线程增加上下文切换的开销
- I/O 密集型
  - 线程数可以适当增大，具体根据任务类型（如网络请求、文件 I/O）调整

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor() as executor:
    print(executor._max_workers)  # 输出默认线程数
```

## 线程池

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor() as executor:
    # 提交任务
    future1 = executor.submit(pow, 323, 1235)
    future2 = executor.submit(pow, 100, 2)
    # 获取任务结果
    print(future1.result())
    print(future2.result())

    # 返回结果迭代器
    res = executor.map(pow, [323, 1235], [1235, 1235])
```

## map 函数 & lambda 函数

### lambda 函数

```python
# 嵌套 lambda 函数
# lambda 函数解包参数
lambda item: (lambda route, flag: func("", route, flag))(*item)

# item也可以是字典等可迭代对象
# 示例：解包字典
kwargs = {"x": 1, "y": 2, "z": 3}
result = (lambda x, y, z: x + y + z)(**kwargs)  # 使用 ** 解包字典
print(result)  # 输出: 6

def func(key1, key2):
    return key1 + key2
result = (lambda key1, key2: func(key1, key2))(**item)

# 示例：同时解包列表和字典
args = [1, 2]
kwargs = {"z": 3}
result = (lambda x, y, z: x + y + z)(*args, **kwargs)
print(result)  # 输出: 6

# 示例：嵌套解包
nested = [{"x": 1, "y": 2, "z": 3}, {"x": 4, "y": 5, "z": 6}]
result = list(map(lambda item: (lambda x, y, z: x + y + z)(**item), nested))
print(result)  # 输出: [6, 15]
```

### map 函数

```python
map(func, iterables)


def add(x, y):
    return x + y

list1 = [1, 2, 3]
list2 = [4, 5, 6]

# 使用 map 将两个列表中的元素传入 add 函数
result = map(add, list1, list2)
print(list(result))  # 输出: [5, 7, 9]

list1 = [10, 20, 30]
list2 = [1, 2, 3]

# 使用 lambda 表达式直接传递两个值
result = map(lambda x, y: x * y, list1, list2)
print(list(result))  # 输出: [10, 40, 90]

def concat(prefix, word):
    return prefix + word

# 第一个参数固定为 "", 第二个参数是列表
words = ["apple", "banana", "cherry"]

# 使用 map 函数
result = map(lambda word: concat("", word), words)
print(list(result))  # 输出: ['apple', 'banana', 'cherry']

# 使用 itertools.repeat
from itertools import repeat

# 定义列表
words = ["apple", "banana", "cherry"]

# 使用 repeat 将 "" 作为固定值传递
result = map(concat, repeat(""), words)
print(list(result))  # 输出: ['apple', 'banana', 'cherry']
```
