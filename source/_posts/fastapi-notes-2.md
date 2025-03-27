---
title: FastAPI Notes 2
date: 2024-12-11 16:25:20
tags: python
categories: posts
---

# FastAPI Notes 2

## 关闭交互式文档访问

```python
from fastapi import FastAPI
app = FastAPI(
    docs_url=None, 
    redoc_url=None,
    # 或直接设置
    openapi_url=None)
```

## 全局异常错误捕获
```python
from fastapi import FastAPI
from starlette.respnses import JSONResponse
async def exception_not_found(request, exc):
    return JSONResponse({
        "code": exc.status_code,
        "error": "not found",
        }, status_code=exc.status_code)

exception_handlers = {
    404: exception_not_found,
    }

app = FastAPI(
    exception_handlers=exception_handlers)
```

## 多应用挂载

### 主从应用挂载

```python
from fastapi import FastAPI
from fastapi.repsponses import JSONResponse

app = FastAPI(title="主应用")
@app.get("/index")
async def index():
    return JSONResponse({"msg": "主应用"})

subapp = FastAPI(title="子应用")
@subapp.get("/index")
async def index():
    return JSONResponse({"msg": "子应用"})

app.mount(path="/subapp", app=subapp, name="子应用")
```
- 主应用交互式文档地址为http://127.0.0.1:8000/docs。
- 子应用交互式文档地址为http://127.0.0.1:8000/subapp/docs

### 挂载其他WSGI应用

可以用来挂载如Flask、Django等应用。

```python
# 创建fastapi主应用
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()
@app.get("/index")
async def index():
    return JSONResponse({"msg": "fastapi应用"})


from flask import Flask
from fastapi.middleware.wsgi import WSGIMiddleware
flask_app = Flask(__name__)
@flask_app.route("/index")
def flask_main():
    return {"index": "flask应用"}

app.mount(path="/flaskapp", app=WSGIMiddleware(flask_app), name="flask应用")
```

## 自定义配置swagger_ui

由于相关的swagger-ui.css和swagger-ui-bundle.js资源是从第三方的CDN服务商上加载的，第三方的CDN服务出现问题就会导致无法正常加载可视化API文档界面的情况。为了避免出现这种情况，以及在无网络情况下也能正常访问API交互式文档，就需要自定义（或改造）并渲染HTML模板中的一些变量，让swagger-ui.css和swagger-ui-bundle.js等静态资源从本地进行加载。


```python
from fastapi import FastAPI
from fastapi.openapi.docs import get_swagger_ui_html, get_redoc_html, get_swagger_ui_oauth2_redirect_html
from fastapi.staticfiles import StaticFiles
import pathlib

app = FastAPI(docs_url=None, redoc_url=None)
app.mount(
    path="/static",
    app=StaticFiles(directory=f"pathlib.Path.cwd()/static"),
    name="static"
)

# 函数参数缺一不可，否则会访问不了
@app.get("/docs", include_in_schema=False)
async def custom_swagger_ui_html():
    return get_swagger_ui_html(
        openapi_url=app.openapi_url,
        title=app.title + " - Swagger UI",
        oauth2_redirect_url=app.swagger_ui_oauth2_redirect_url,
        swagger_js_url="/static/swagger-ui-bundle.js",
        swagger_css_url="/static/swagger-ui.css",
        swagger_favicon_url="/static/favicon.png",
    )
```

## Body参数

### 引入Pydantic模型声明请求体

将请求体参数识别为JSON格式字符串，并自动将字段转换为相应的数据类型。

自动进行参数规则的校验。如果校验失败，则响应报文内容会自动返回一个错误，并准确指出错误数据的位置和信息。

为模型生成JSON Schema定义，并显示在API交互式文档中，Schema会成为OpenAPI Schema的一部分。

在函数内部，可以直接访问模型对象的所有属性。

```python
from pydantic import BaseModel
from typing import Optional
class User(BaseModel):
    username: str = Field(..., min_length=1, max_length=20, title="用户名", description="用户名", example="admin")
    password: str = Field(..., min_length=8, max_length=20)
    age: Optional[int] = None

# 把模型绑定到视图函数中
@app.post("/user")
async def create_user(user: User):
    return user
```

### 用Body声明请求体

```python
@app.post("/user")
async def create_user(
    username: str = Body(..., min_length=1, max_length=20),
    password: str = Body(..., min_length=8, max_length=20),
    age: Optional[int] = Body(None, gt=0, lt=100),
    ):
    return {"username": username, "password": password, "age": age}

```

### Body的embed参数

```python
class User(BaseModel):
    username: str
    password: str
    age: Optional[int] = None

@app.post("/user")
async def create_user(user: User = Body(..., embed=True)):
    return user
```
## 后台异步任务执行

这种后台执行的任务本质是在当前服务进程中完成的，若当前服务进程被关闭，则任务也会被终止

```python
# 创建后台任务函数
def background_task(n):
    time.sleep(n)
# 定义后台任务
from fastapi import BackgroundTasks
@app.api_route(path="/task", methods=["POST","GET"])
async def create_task(background_tasks: BackgroundTasks):
    background_tasks.add_task(background_task, 3)
    return {"msg": "success"}
```
## 应用启动和关闭事件

startup和shutdown事件是FastAPI提供的进行服务启动和关闭时执行的事件回调通知处理机制

1. startup事件

应用场景：
- 设置和读取应用配置参数。
- 通过对数据库初始化把初始化的对象存储到app对象上下文中。
- 初始化第三方插件

```python
app = FastAPI()
@app.on_event("startup")
async def startup_event():
    print("应用启动")
```

2. shutdown事件

当服务关闭并进行一些资源释放操作时，可以通过shutdown事件进行相关资源释放处理

只有当所有连接都已关闭，并且任何正在进行的后台任务都已完成时，才会调用关闭处理事件。如果服务被强制进行kill操作，则回调函数不会执行。

```python
@app.on_event("shutdown")
async def shutdown_event():
    print("应用关闭")
```
## HTTPException 异常处理

通过@app.exception_handler(HTTPException)进行一个全局异常的捕获处理。当手动抛出HTTPException异常时，FastAPI会对此类异常进行拦截处理，并进入http_exception_handler()函数。在该函数内部会对异常信息进行提取处理，然后通过JSONResponse进行响应报文封装输出并返回。

```python
from fastapi import HTTPException, FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi import Query
app = FastAPI()

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        content=exc.detail,
        status_code=exc.status_code,
        headers=exc.headers)
```

## 设置和环境变量

```ini
# .env文件设置
ADMIN_EMAIL="deadpool@example.com"
APP_NAME="ChimichangApp"
```

```python
from functools import lru_cache
from pydantic_settings import BaseSettings
from dotenv import load_dotenv
class Settings(BaseSettings):
    app_name: str = "Awesome API"
    admin_email: str
    items_per_user: int = 50

    class Config:
        env_file = ".env"

@lru_cache
def get_settings():
    load_dotenv()
    return Settings()
```

## 中间件记录请求日志

```python
from fastapi import FastAPI, Request
import time

app = FastAPI()

# 定义中间件函数
async def log_request_info(request: Request, call_next):
    start_time = time.time()
    # 获取客户端IP地址（考虑反向代理情况）
    x_forwarded_for = request.headers.get("X-Forwarded-For")
    if x_forwarded_for:
        client_ip = x_forwarded_for.split(",")[0].strip()
    else:
        client_ip = request.client.host

    # 获取请求方法（如GET、POST等）
    request_method = request.method
    # 获取请求路径
    request_path = request.url.path
    # 获取用户代理信息
    user_agent = request.headers.get("user-agent")

    try:
        response = await call_next(request)
    finally:
        process_time = (time.time() - start_time) * 1000
        print(f"[Request Log] - IP: {client_ip}, Method: {request_method}, Path: {request_path}, User-Agent: {user_agent}, Process Time: {process_time:.2f}ms")

    return response

# 添加中间件到应用
app.middleware("http")(log_request_info)

# 定义一个简单的路由用于测试
@app.get("/")
async def read_root():
    return {"message": "Hello, World!"}
```


## 文件上传

### UploadFile

UploadFile 是一个异步对象，它的文件内容是通过 read() 方法逐步读取的。一旦你调用 read() 方法并读取文件内容，文件指针会被移动，后续的读取会发生错误或无法读取到任何内容。这意味着如果你在一个函数中调用了 read() 并读取了文件的内容，那么在另一个函数中调用 read() 时，文件指针已经到达文件的末尾，无法再次读取文件内容。

因此可以使用变量对读取的内容进行保存来多次使用，而不是多次调用 read() 方法。


### 接收文件上传

```python
from fastapi import FastAPI, File, UploadFile
from typing import List

app = FastAPI()

@app.post("/upload-multiple/")
async def upload_multiple_files(files: List[UploadFile] = File(...)):
    file_details = []
    for file in files:
        content = await file.read()
        print(f"Filename: {file.filename}, Content-Type: {file.content_type}")
        file_details.append({"filename": file.filename, "content_type": file.content_type})
    return {"files": file_details}

## 保存上传的文件
from fastapi import FastAPI, File, UploadFile

app = FastAPI()

@app.post("/save-file/")
async def save_file(file: UploadFile = File(...)):
    file_location = f"uploads/{file.filename}"  # 文件保存路径
    with open(file_location, "wb") as buffer:
        buffer.write(await file.read())  # 保存文件内容
    return {"info": f"File '{file.filename}' saved at '{file_location}'"}
```

### 限制上传文件大小

```python
from fastapi import FastAPI, Request, HTTPException
from starlette.middleware.trustedhost import TrustedHostMiddleware
from starlette.middleware.exceptions import ExceptionMiddleware
from starlette.middleware.gzip import GZipMiddleware

app = FastAPI()

# 设置文件大小限制
@app.middleware("http")
async def limit_upload_size(request: Request, call_next):
    max_size = 1024 * 1024  # 最大文件大小：1MB
    content_length = request.headers.get("Content-Length")
    if content_length and int(content_length) > max_size:
        raise HTTPException(status_code=413, detail="File too large")
    return await call_next(request)
```