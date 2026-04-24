---
title: Makefile 编写教程
date: 2026-04-24 10:36:21
categories: Notes
tags:
  - Linux
  - CICD
  - Build
---

## Makefile 是什么

Makefile 是一个构建自动化工具，由 `make` 程序读取并执行。它定义了文件之间的依赖关系和构建规则，只重新编译发生变化的部分，避免全量构建。

核心思想：**根据文件的时间戳判断哪些目标需要更新**。

## 基本语法

Makefile 由一系列**规则（rule）**组成：

```makefile
target: prerequisites
	recipe
```

- **target**：目标文件或伪目标名
- **prerequisites**：依赖文件列表，空格分隔
- **recipe**：构建命令，**必须以 Tab 缩进**（不能用空格）

### 第一个例子

```makefile
# 编译单个 C 程序
hello: hello.o
	gcc -o hello hello.o

hello.o: hello.c
	gcc -c hello.c

clean:
	rm -f hello hello.o
```

执行：

```bash
make        # 构建第一个目标 hello
make clean  # 执行 clean 目标
```

## 变量

变量让 Makefile 更易维护，避免重复。

### 定义与使用

```makefile
CC = gcc
CFLAGS = -Wall -O2
TARGET = myapp
SRCS = main.c utils.c

$(TARGET): $(SRCS)
	$(CC) $(CFLAGS) -o $(TARGET) $(SRCS)
```

### 两种赋值

```makefile
# 递归展开（使用时才求值，可能产生递归）
VAR = $(OTHER)

# 简单展开（定义时就确定值，推荐）
VAR := $(OTHER)
```

**推荐使用 `:=`**，行为可预测，性能更好。

```makefile
# := 示例
A := hello
B := $(A) world   # B = "hello world"
A := goodbye       # B 仍然是 "hello world"

# = 示例
A = hello
B = $(A) world     # B 的值在展开时才确定
A = goodbye         # B 展开后变成 "goodbye world"
```

### 追加变量

```makefile
CFLAGS = -Wall
CFLAGS += -O2      # CFLAGS 现在是 "-Wall -O2"
```

### 命令行覆盖

用 `?=` 设置默认值（仅在未定义时生效）：

```makefile
CC ?= gcc
```

## 自动变量

在 recipe 中可用的快捷变量：

| 变量 | 含义 |
|------|------|
| `$@` | 当前目标名 |
| `$<` | 第一个依赖文件 |
| `$^` | 所有依赖文件（去重） |
| `$?` | 比目标新的依赖文件 |
| `$*` | 匹配模式规则中 `%` 的部分 |

```makefile
app: main.o utils.o
	gcc -o $@ $^

main.o: main.c main.h
	gcc -c $< -o $@
```

## 模式规则

用 `%` 作为通配符，批量定义同类规则：

```makefile
# 所有 .c 文件编译为 .o 文件
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```

一次定义覆盖所有 `.c → .o` 的编译，无需逐个写规则。

## 常用内建函数

Make 提供了丰富的文本处理函数，调用格式：`$(function arguments)`。

### 字符串处理

```makefile
# 替换：将 SRCS 中的 .c 替换为 .o
OBJS = $(patsubst %.c, %.o, $(SRCS))

# 去除首尾空格
STRIPPED = $(strip   hello world   )

# 查找字符串
FOUND = $(findstring hello, hello world)   # 返回 hello
```

### 文件名操作

```makefile
SRCS = src/main.c lib/utils.c

# 获取目录部分
DIRS = $(dir $(SRCS))           # src/ lib/

# 获取文件名部分
FILES = $(notdir $(SRCS))       # main.c utils.c

# 去除后缀
BASES = $(basename $(SRCS))     # src/main lib/utils

# 添加后缀
OBJS = $(addsuffix .o, $(BASES))  # src/main.o lib/utils.o

# 添加前缀
PATHS = $(addprefix build/, $(FILES))  # build/main.c build/utils.c

# 通配符匹配
WILDCARDS = $(wildcard src/*.c)
```

### 条件与循环

```makefile
# 过滤
C_FILES = $(filter %.c, $(SRCS))        # 只保留 .c 文件
NON_C = $(filter-out %.c, $(SRCS))      # 排除 .c 文件

# 遍历（对每个元素执行操作）
OBJS = $(foreach src, $(SRCS), $(src:.c=.o))
```

## 伪目标

伪目标不代表实际文件，总是会被执行。用 `.PHONY` 声明：

```makefile
.PHONY: clean install all

clean:
	rm -f *.o $(TARGET)

install: $(TARGET)
	cp $(TARGET) /usr/local/bin/

all: $(TARGET)
```

不声明 `.PHONY` 时，如果目录下恰好存在同名文件 `clean`，`make clean` 会认为目标已是最新而跳过。

## 条件判断

```makefile
# 根据条件选择不同行为
ifdef DEBUG
	CFLAGS += -g -DDEBUG
else
	CFLAGS += -O2
endif

# 判断变量是否等于某值
ifeq ($(OS), Windows_NT)
	EXE_EXT = .exe
else
	EXE_EXT =
endif
```

## 自动依赖生成

手动维护头文件依赖很麻烦。让编译器自动生成：

```makefile
SRCS = main.c utils.c
OBJS = $(SRCS:.c=.o)
TARGET = app

# 编译时同时生成 .d 依赖文件
%.o: %.c
	$(CC) $(CFLAGS) -MMD -c $< -o $@

# 包含所有 .d 文件（首次构建时 .d 不存在，用 -include 忽略错误）
-include $(SRCS:.c=.d)
```

`-MMD` 让 gcc 生成 `.d` 文件，里面记录了每个 `.o` 依赖哪些头文件。`-include` 前面的 `-` 表示文件不存在时不报错。

## 实战：完整 C 项目模板

```makefile
CC      := gcc
CFLAGS  := -Wall -Wextra -O2 -MMD
LDFLAGS :=
LDLIBS  :=

SRCS    := $(wildcard src/*.c)
OBJS    := $(SRCS:.c=.o)
DEPS    := $(OBJS:.o=.d)
TARGET  := myapp

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(LDFLAGS) -o $@ $^ $(LDLIBS)

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(DEPS) $(TARGET)

-include $(DEPS)
```

## 实战：非 C 项目的 Makefile

Makefile 不只用于 C 项目，任何需要自动化构建的场景都适用。

### Go 项目

```makefile
.PHONY: build test clean

BINARY := myapp
GO     := go

build:
	$(GO) build -o $(BINARY) ./cmd/app

test:
	$(GO) test ./... -v

clean:
	rm -f $(BINARY)

run: build
	./$(BINARY)
```

### Docker 部署

```makefile
.PHONY: build up down logs

IMAGE := myapp
TAG   := latest

build:
	docker build -t $(IMAGE):$(TAG) .

up:
	docker compose up -d

down:
	docker compose down

logs:
	docker compose logs -f

push: build
	docker push $(IMAGE):$(TAG)
```

### 文档/静态站点

```makefile
.PHONY: serve build clean

serve:
	hugo server -D

build:
	hugo --minify

clean:
	rm -rf public/

deploy: build
	rsync -avz public/ server:/var/www/site/
```

## 常见陷阱

**Tab 和空格**：recipe 行必须用 Tab 缩进。编辑器可能将 Tab 转为空格，导致报错 `missing separator`。解决方法：编辑器设置 Make 文件类型下保留 Tab。

**变量在 recipe 中展开的时机**：recipe 中的变量在执行时展开，不是在定义时。需要立即求值时用 `$$` 转义 shell 变量：

```makefile
# 错误：$SHELL 会被 Make 解释
print-shell:
	echo $(SHELL)

# 正确：$$ 传递给 shell
print-shell:
	echo $$SHELL
```

**命令在独立 shell 中执行**：每行 recipe 是独立的 shell 进程，`cd` 不会跨行生效：

```makefile
# 错误：第二行不在 build 目录下
build:
	mkdir -p build
	cd build
	cmake ..

# 正确：用 && 连接或反斜杠续行
build:
	mkdir -p build && cd build && cmake ..
```

## 调试技巧

```bash
# 不实际执行，只打印命令（干运行）
make -n

# 忽略错误继续执行
make -i

# 并行构建（4 个任务）
make -j4

# 调试模式，显示变量展开和规则搜索过程
make -d

# 打印某个变量的值
make print-VAR
# 配合：
# print-%:
# 	@echo $* = $($*)
```
