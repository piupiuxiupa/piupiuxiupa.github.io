---
title: FastAPI Notes
date: 2024-11-20 16:36:58
tags: python
---

# FastAPI Notes

> FastAPI 具有python的一些新特性，比如类型提示，异步支持，以及自动生成文档。

## 模块
- fastapi
  - FastAPI
  - Path
  - Query
  - Body
  - Cookie
- pydantic
  - BaseModel
- typing
  - Annotated
  - Union
  - Optional
  - List
  - Set
- starlette
  - RedirectResponse

## 简单例子

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
async def root():
    return {"message": "Hello World"}
```

##  启动命令

uvicorn main:app 命令含义如下:

main：main.py 文件（一个 Python「模块」）。
app：在 main.py 文件中通过 app = FastAPI() 创建的对象。
--reload：让服务器在更新代码后重新启动。仅在开发时使用该选项。

```bash
uvicorn main:app --reload
```

## 查看文档

- http://127.0.0.1:8000/docs
- http://127.0.0.1:8000/redoc

### 查看文档显示空白问题

因为fastapi的openapi包里的docs.py文件接口文档默认使用

https://cdn.jsdelivr.net/npm/swagger-ui-dist@5.9.0/swagger-ui.css

https://cdn.jsdelivr.net/npm/swagger-ui-dist@5.9.0/swagger-ui-bundle.js

国外地址访问较慢，如果是纯内网环境也是如此。

```python
# 替换fastapi下openapi包内的docs.py文件的js和css参数的值
# 将url替换为本地js和css文件
# Lib/site-packages/fastapi/openapi/docs.py
# 修改其中get_swagger_ui_html 函数
# redoc则修改get_redoc_html函数
swagger_js_url: str="/static/swagger-ui/swagger-ui-bundle.js",
swagger_css_url: str="/static/swagger-ui/swagger-ui.css",
swagger_favicon_url: str="/static/swagger-ui/favicon.png",

redoc_js_url: str = "/static/redoc/bundles/redoc.standalone.js",
redoc_favicon_url: str = "/static/redoc/favicon.png"

# 修改完，然后挂载， directory 填本地路径
app.mount('/static', StaticFiles(directory='static'),
          name='static')	
```


## FastAPI 参数

> FastAPI 是直接从 Starlette 继承的类。
>
> 你可以通过 FastAPI 使用所有的 Starlette 的功能。

- **路径**

  这里的「路径」指的是 URL 中从第一个 / 起的后半部分。

  所以，在一个这样的 URL 中：

  `https://example.com/items/foo`
    
  ...路径会是：

  /items/foo

- **操作**

  在开发 API 时，通常使用特定的 HTTP 方法去执行特定的行为。

  通常使用：

  POST：创建数据。

  GET：读取数据。

  PUT：更新数据。

  DELETE：删除数据。

**路径操作**是按顺序依次运行的

下面例子先运行`/users/me`，如果位置互换，会先运行`/users/{user_id}`，导致`/users/me`失效。
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/users/me")
async def read_user_me():
    return {"user_id": "the current user"}

@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"user_id": user_id}
```

**包含路径的路径参数**

假设**路径操作**的路径为 `/files/{file_path}`。

但需要 file_path 中也包含**路径**，比如，`home/johndoe/myfile.txt`。

此时，该文件的 URL 是这样的：`/files/home/johndoe/myfile.txt`。

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    return {"file_path": file_path}
```

**查询参数**

声明的参数不是路径参数时，路径操作函数会把该参数自动解释为**查询**参数。

查询字符串是键值对的集合，这些键值对位于 URL 的 ? 之后，以 & 分隔。

例如，以下 URL 中：
`http://127.0.0.1:8000/items/?skip=0&limit=10`

```python
from fastapi import FastAPI

app = FastAPI()

fake_items_db = [{"item_name": "Foo"}, {"item_name": "Bar"}, {"item_name": "Baz"}]

@app.get("/items/")
async def read_item(skip: int = 0, limit: int = 10):
    return fake_items_db[skip : skip + limit]
```

**可选参数**

把默认值设为 None 即可声明**可选的**查询参数：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
## str | None 表示 q 可以是一个字符串类型的值，也可以是 None
async def read_item(item_id: str, q: str | None = None):
    if q:
        return {"item_id": item_id, "q": q}
    return {"item_id": item_id}
```
> FastAPI 可以识别出 item_id 是路径参数，q 不是路径参数，而是查询参数。
>
> 如果要把查询参数设置为**必选**，就不要声明默认值.
>
> 如果只想把参数设为**可选**，但又不想指定参数的值，则要把默认值设为 None

**请求体**

**路径**中声明了相同参数的参数，是路径参数
类型是（int、float、str、bool 等）**单类型**的参数，是**查询**参数
类型是 Pydantic 模型的参数，是请求体

```python
from fastapi import FastAPI

## 导入 Pydantic 的 BaseModel
from pydantic import BaseModel

# 创建数据模型
class Item(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

app = FastAPI()

@app.put("/items/{item_id}")
# 声明请求体参数 item: Item
# 路径参数 + 请求体 + 查询参数
async def update_item(item_id: int, item: Item, q: str | None = None):
    result = {"item_id": item_id, **item.dict()}
    if q:
        result.update({"q": q})
    return result
```

**查询参数和字符串校验**

```python
from typing import Union
# 导入Query
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
# 使用Query作为默认值
async def read_items(
    # 添加最小长度、最大长度、正则表达式等校验
    # Optional[str] 等价于 Union[str, None]，但更具可读性
    q: Union[str, None] = Query(default=None, min_length=3, max_length=50, pattern="^fixedquery$")
):
    results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
    if q:
        results.update({"q": q})
    return results
```
> 当你在使用 Query 且需要声明一个值是必需的时，只需不声明默认参数即可。
>
> 也可使用省略号`...`声明必须参数，如 `read_items(q: str = Query(default=..., min_length=3)):`

**使用None类型时声明必须参数**

声明一个参数可以接收`None`，但仍然是必需的。

```python
# 声明 None是有效类型，使用 ...
# 或使用Pydantic中的Required代替省略号(...)
async def read_items(q: Union[str, None] = Query(default=..., min_length=3)):
```
> 大多数情况下，当你需要某些东西时，可以简单地省略 default 参数，因此你通常不必使用 ... 或 Required

**查询参数列表 / 多个值**

> 要声明类型为 list 的查询参数，如上例所示，你需要显式地使用 Query，否则该参数将被解释为请求体。

```python
from typing import List, Union

from fastapi import FastAPI, Query

app = FastAPI()

## http://localhost:8000/items/?q=foo&q=bar
@app.get("/items/")
# List[str] 会检查列表内容必须为字符串
# 使用list则不会检查
# async def read_items(q: list = Query(default=[])):
async def read_items(q: Union[List[str], None] = Query(default=None)):
# 有多个值的默认值 
# async def read_items(q: List[str] = Query(default=["foo", "bar"])):
    query_items = {"q": q}
    return query_items
```
**声明更多元数据**

你可以添加更多有关该参数的信息。

这些信息将包含在生成的 OpenAPI 模式中，并由文档用户界面和外部工具所使用。
```python
from typing import Union

from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
async def read_items(
    q: Union[str, None] = Query(
        default=None,
        title="Query string",
        description="Query string for the items to search in the database that have a good match",
        min_length=3,
    ),
):
    results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
    if q:
        results.update({"q": q})
    return results
```

**路径参数和数值校验**

```python
from fastapi import FastAPI, Path, Query

app = FastAPI()


@app.get("/items/{item_id}")
async def read_items(
    *,
    item_id: Annotated[int, Path(title="The ID of the item to get", ge=0, le=1000)],
    q: Annotated[str | None, Query(alias="item-query")] = None,
    size: float = Query(gt=0, lt=10.5),
    
):
    results = {"item_id": item_id}
    if q:
        results.update({"q": q})
    if size:
        results.update({"size": size})
    return results
```

**请求体内声明校验和元数据**
```python
from typing import Annotated, List

from fastapi import Body, FastAPI
from pydantic import BaseModel, Field, HttpUrl

app = FastAPI()
class Image(BaseModel):
  # 将被检查是否为有效的 URL
    url: HttpUrl
    name: str

class Item(BaseModel):
    name: str
    description: str | None = Field(
        default=None, title="The description of the item", max_length=300
    )
    price: float = Field(gt=0, description="The price must be greater than zero")
    tax: float | None = None
    tags: List[str] = []
    Image: Image | None = None  
    # images: list[Image] | None = None
    # tags: set[str] = set()

@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Annotated[Item, Body(embed=True)]):
    results = {"item_id": item_id, "item": item}
    return results

```

## Cookie, Header

Header 比 Path、Query 和 Cookie 提供了更多功能

大部分标准请求头用**连字符**分隔，即**减号**（-）。

但是 user-agent 这样的变量在 Python 中是无效的。

因此，默认情况下，Header 把参数名中的字符由下划线（_）改为连字符（-）来提取并存档请求头 。

同时，HTTP 的请求头不区分大小写，可以使用 Python 标准样式（即 snake_case）进行声明

如需禁用下划线自动转换为连字符，可以把 Header 的 convert_underscores 参数设置为 False

```python
from typing import Annotated

from fastapi import Cookie, FastAPI, Header

app = FastAPI()


@app.get("/items/")
async def read_items(ads_id: Annotated[str | None, Cookie()] = None, user_agent: Annotated[str | None, Header()] = None):
    return {"ads_id": ads_id, "user_agent": user_agent}
```

## 响应模型
- response_model
  - 定义响应模型，特别是确保私有数据被过滤掉
- response_model_exclude_unset
  - 响应中不会包含默认值，而是仅有实际设置的值，即仅返回显式设定的值