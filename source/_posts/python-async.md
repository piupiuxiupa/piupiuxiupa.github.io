---
title: Python 异步
date: 2024-12-27 20:43:35
tags: python
categories: Posts
---

# Python 异步编程

使用 async 和 await 时，await 会使当前的协程暂停并等待异步操作完成。

```python
import asyncio

async def async_task(i):
    print(f"Task {i} started")
    await asyncio.sleep(1)
    print(f"Task {i} completed")

async def main():
    for i in range(3):
        await async_task(i)

# 运行协程
asyncio.run(main())
```

执行过程：

1. Task 0 started 会打印出来。
2. await asyncio.sleep(1) 会暂停当前协程 1 秒，并将控制权交回事件循环。
3. 在这 1 秒钟内，事件循环可能会运行其他任务（如果有的话）。
4. 1 秒后，Task 0 completed 会打印出来，然后循环进入下一次迭代，Task 1 开始。

## 异步并行任务

```python
async def main():
    tasks = [async_task(i) for i in range(3)]
    await asyncio.gather(*tasks)
```

1. 返回值：每个异步任务的返回值会按任务的顺序排列在 gather() 返回的列表中。
2. 捕获异常：asyncio.gather() 默认会在某个任务抛出异常时取消其他任务的执行。如果想捕获所有任务的异常并继续执行，可以将 return_exceptions=True 传递给 gather()。

async def 定义的函数返回一个协程对象，只有通过 await 或通过事件循环运行它时，它才会开始执行。

- async_task(i) 返回一个协程对象，并没有立刻执行任务，只是生成了一个协程。
- await asyncio.gather(*tasks) 才是实际调度这些协程并开始执行它们的代码。

```python
import asyncio

async def async_task(i):
    print(f"Task {i} started")
    await asyncio.sleep(1)
    return f"Result from Task {i}"

async def main():
    # 创建协程对象列表，但这些协程对象并没有开始执行
    tasks = [async_task(i) for i in range(3)]
    print("Tasks created")

    # 通过 asyncio.gather 来运行所有协程对象并等待它们的结果
    results = await asyncio.gather(*tasks)
    
    print("All results:", results)

# 运行协程
asyncio.run(main())
```

## aiohttp

```python
import aiohttp
import asyncio

# 异步函数，发送 HTTP 请求
async def fetch(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()

# 调用异步函数
async def main():
    url = "https://jsonplaceholder.typicode.com/todos/1"
    result = await fetch(url)
    print(result)

# 启动事件循环
if __name__ == "__main__":
    asyncio.run(main())
```

> aiohttp 的 headers、URL 参数、cookies 中的 键 必须是字符串类型
> 
> aiohttp 不接受 None 类型的数据

## aiofiles

```python
import aiofiles
import asyncio

# 异步写文件
async def write_file():
    async with aiofiles.open('example.txt', mode='w') as f:
        await f.write("Hello, aiofiles!")

# 异步读文件
async def read_file():
    async with aiofiles.open('example.txt', mode='r') as f:
        content = await f.read()
        print(content)

async def main():
    await write_file()  # 写入文件
    await read_file()   # 读取文件

if __name__ == "__main__":
    asyncio.run(main())
```

