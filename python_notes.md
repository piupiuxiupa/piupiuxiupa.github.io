# Python Notes

## isinstance函数

```python
# 判断类型
if isinstance(url, str):
  # 字符串转列表会把字符串打散成字母列表存放
  # 可以如下方法转成列表存放
  url = [url]
  # 如果不传递任何分隔符给split()方法，且字符串中没有空格（默认分隔符）或其他空白字符
  # 字符串将不会被拆分，而是作为一个整体放入列表中。
  url = url.split()
if isinstance(urls, list):
  # append是在列表末尾添加元素，会变成列表嵌套
  # extend 在列表末尾添加元素，会将列表合并成一个列表
  res.extend(urls)

# 判断对象是否是某个类或子类的实例
# 假设 error是一个异常对象 requests.exceptions.MissingSchema
isinstance(error, requests.exceptions.MissingSchema)
# 这里Exception也可以判断，Exception是所有异常类的基类
isinstance(error, Exception)
```

## JSON

python中字典跟json很相似，但是字典类型是字典，json是字符串

```python
data = {"name": "nobody"} # 此时是字典类型
data = json.dumps(data) # 转换后，data就是字符串类型
```

## None 和 ''
None并不完全等于''
None 表示空值或缺失值，而 '' 表示空字符串
```python
# 空字符串是False
if not '':
  print("空字符串")
```


## Flask记录日志

```python
import logging
from flask import Flask
app = Flask(__name__)
app.logger.setLevel(logging.DEBUG)
# StreamHandler，它会将日志输出到控制台。可以通过addHandler方法将这个处理程序添加到 Flask 应用的日志记录器中。
console_handler = logging.StreamHandler()
app.logger.addHandler(console_handler)

# 将日志记录到文件中，可以使用FileHandler
file_handler = logging.FileHandler('app.log')
# INFO级别及以上的日志会被记录到文件中
file_handler.setLevel(logging.INFO)
app.logger.addHandler(file_handler)

# RotatingFileHandler会将日志记录到app.log文件中，当文件大小达到1MB（maxBytes=1024*1024）时，会自动轮转日志文件，最多保留3个备份文件（backupCount=3），并且会记录DEBUG级别及以上的日志。
from logging.handlers import RotatingFileHandler
rotating_handler = RotatingFileHandler('app.log', maxBytes=1024*1024, backupCount=3)
# 可以给每个handler单独设置日志等级
rotating_handler.setLevel(logging.DEBUG)
app.logger.addHandler(rotating_handler)

formatter = Formatter(" %(asctime)s threadId-%(thread)d %(levelname)s %(module)s %(funcName)s %(message)s")
rotating_handler.setFormatter(formatter)

# 配置TimedRotatingFileHandler保留策略
time_handler = TimedRotatingFileHandler(f'{LogPath}/main.log',
                                            when="midnight",
                                            interval=1,
                                            backupCount=20,
                                            encoding='utf-8')
time_handler.setFormatter(formatter)

@app.route('/')
def index():
    app.logger.debug("这是一条调试信息")
    app.logger.info("这是一条普通信息")
    app.logger.warning("这是一条警告信息")
    app.logger.error("这是一条错误信息")
    app.logger.critical("这是一条严重错误信息")
    return "日志记录示例"
```

## 读取sql文件内容，并用pymysql执行

```python
# 假设读取了一段 sql
sql_text: str = sql_text
# 对 sql内容进行处理，变为 pymysql可以执行的形式
sqls: list[str] = sql.replace('\n','').split(';')
# 连接数据库
conn = pymysql.connect(host='localhost', user='root', password='password', database='mydatabase')
cursor = conn.cursor()
for sql in sqls:
  cursor.execute(sql)
conn.commit()
cursor.close()
conn.close()
```
## eval构造代码执行

> 需要防止注入

```python
# DB_Info 对象
sql_str= f"DB_Info.query.filter_by(**keys)"

for v in keys_list:
  sql_str += f".filter(DB_Info.{v}.in_(data['{v}']))"

payload = f"eval({sql_str})"
db_info = eval(sql_str)
```
## sqlalchemy or_

在 SQLAlchemy 中，or_ 函数用于创建 OR 条件。它允许你在查询中使用多个条件，其中任何一个条件满足都会返回结果。

```python
from sqlalchemy import or_

Obj.query.filter(or_
                 (
                 or_(Obj.name == 'name', Obj.name == None),
                 or_(Obj.age == 18, Obj.age == 19)
                 )
                 ).all()
```

## flask_sqlalchemy

```python
# filter_by 可以接受 mapping 类型的参数
Obj.query.filter_by(name='name', age=18).all()
# data 为字典类型
Obj.query.filter_by(**data).all()
```

## 类型注解

```python
from typing import Union

## 表示 list里的值包含 str和 int两种类型
test: list[Union[str, int]] = []
```

## Flask content-type
```python
# 可以用来判断请求的 content-type
request.headers.get('Content-Type') == "application/json"  
request.content_type == "application/json"
```

## functools.wraps

在 Flask（以及其他类似的 Web 框架）中，视图函数的名称（通常是函数的__name__属性）被用来作为端点（endpoint）的默认标识符。在没有@wraps的情况下，包装函数的名称（如wrapper）可能会覆盖其他视图函数的端点，导致AssertionError: view function mapping is overwriting an existing endpoint function错误。

当使用@wraps装饰包装函数（wrapper）时，它会保留被装饰函数（原始视图函数）的重要元数据，其中包括函数名（__name__属性）。

```python
from functools import wraps
def log_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Finished {func.__name__}")
        return result
    return wrapper
```

## loguru

```python
from loguru import logger

logger.debug("这是一条调试信息")
logger.info("这是一条普通信息")
logger.warning("这是一条警告信息")
logger.error("这是一条错误信息")

logger.add("app.log")
logger.info("这条信息会同时输出到控制台和app.log文件中")

log_format="{time: YYYY-MM-DD HH:mm:ss} | {level} | {name}:{function}:{line}: {message}"
logger.add("app.log", format=log_format, level="INFO", rotation="1 day", encoding="utf-8")

logger.debug("这条调试信息不会被记录到文件中，因为级别设置为INFO")
logger.info("这条信息会按照指定格式记录到文件中")

"""
循环，rotation，达到指定大小后建新日志。
保留，retention，定期清理。
压缩，compression，压缩节省空间。
"""

logger.add("file_1.log", rotation="500 MB")  # 自动循环过大的文件
logger.add("file_2.log", rotation="12:00")  # 每天中午创建新文件
logger.add("file_3.log", rotation="1 week")  # 一旦文件太旧进行循环

logger.add("file_X.log", retention="10 days")  # 定期清理

logger.add("file_Y.log", compression="zip")  # 压缩节省空间

```

## requests 上传文件

```python
import requests

# 定义上传的目标 URL
url = "https://example.com/upload"

# 打开文件并将其包含在文件字典中
file_path = "path/to/your/file.txt"
files = {
    'file': open(file_path, 'rb')  # 'file' 是服务器设定的接收字段名
}

# 其他表单字段（如果有）
data = {
    'key1': 'value1',
    'key2': 'value2'
}

# 发送 POST 请求
response = requests.post(url, files=files, data=data)

# 打印响应
if response.status_code == 200:
    print("上传成功:", response.text)
else:
    print("上传失败:", response.status_code, response.text)

# 记得关闭文件
files['file'].close()
```
