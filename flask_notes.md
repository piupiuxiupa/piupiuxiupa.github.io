# Flask 初探

> https://dormousehole.readthedocs.io/en/latest/index.html

## python 虚拟环境

```bash


```


## 重要概念
- 路由
- 蓝图
- 装饰器

## 模块
- Flask
- request
- jsonify
- url_for
- redirect
- abort 
- session
- g
- Blueprint
- render_template
- logging

## 项目布局

```
/home/user/Projects/flask-tutorial
├── flaskr/
│   ├── __init__.py
│   ├── db.py
│   ├── schema.sql
│   ├── auth.py
│   ├── blog.py
│   ├── templates/
│   │   ├── base.html
│   │   ├── auth/
│   │   │   ├── login.html
│   │   │   └── register.html
│   │   └── blog/
│   │       ├── create.html
│   │       ├── index.html
│   │       └── update.html
│   └── static/
│       └── style.css
├── tests/
│   ├── conftest.py
│   ├── data.sql
│   ├── test_factory.py
│   ├── test_db.py
│   ├── test_auth.py
├── .venv/
├── pyproject.toml
└── MANIFEST.in
```

.gitignore

```
.venv/

*.pyc
__pycache__/

instance/

.pytest_cache/
.coverage
htmlcov/

dist/
build/
*.egg-info/
```

## flask启动命令
```bash
flask --app flaskr run --debug
```

## 注意事项

- 包名不能与启动的文件名相同
- 蓝图名不能与函数名重复