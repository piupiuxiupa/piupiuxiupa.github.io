---
title: TMUX 基础操作
date: 2024-07-11 15:20:11
tags: tools
---
# TMUX

- 三个概念
  - session - 会话
    - 每运行一次`tmux`会创建一个session
  - window - 窗口
    - 在session内按前缀键后，`c`键创建的就是一个window
  - pane - 窗格
    - 将window进行分割后得到的就是pane

## 基础操作

### 1. 会话外操作

```bash
tmux attach -t 0 ## 进入一个会话
tmux ls ## 查看当前服务器的tmux会话
tmux kill-session -t 0
tmux kill-pane -t 0
tmux kill-window -t 0

tmux list-commands ## 显示tmux所有命令 
tmux show-options -g ## 显示所有选项
```

### 2. 会话内操作

默认前缀键`ctrl+b`

> 以下操作都在某个tmux session中操作，需要先按前缀键

`"` 上下分屏

`%`左右分屏

`c`创建新窗口

`d`暂时退出tmux窗口

`s`列出会话，选择切换会话

> 切换会话也可用`:` ，然后输入以下命令，可tab补全
>
> switch-client -t 0  
>
> attach-session -t 0 

数字0-9切换窗口

`z`放大缩小窗格

`n`切换下一个窗口

`p`切换上一个窗口

`q`显示窗格编号，根据提示输入对应数字切换pane

`o`切换到下一个窗格，方向键亦可

`space`循环切换窗格布局

`&`关闭当前窗口

`[`开启复制模式

`{`将pane布局往前移动

`}`将pane布局往后移动

`?`查看帮助页

## 个性化

> 以下配置将会修改
>
> ​	前缀键为ctrl+x
>
> ​	开启鼠标模式
>
> ​	绑定kjhl为上下左右，类似vim
>
> ​	设置vi风格模式，这样就可以使用`[`复制模式下`/`来查找输出信息的关键字

修改配置文件` ~/.tmux.conf`

```bash
set -g prefix C-x
unbind C-b
bind C-x send-prefix
#set swap pane key
bind-key k select-pane -U
bind-key j select-pane -D
bind-key h select-pane -L
bind-key l select-pane -R
set-option -g mouse on
setw -g mode-keys vi
```

修改完配置文件后关掉session，重开就能生效了
```bash
## 或者也可用此命令生效
tmux source-file ~/.tmux.conf
```

部分终端修改后可能无法复制，需要安装`xclip`，再修改配置文件

```bash
apt install xclip
```

```bash
# 启用鼠标支持
set -g mouse on

# 允许鼠标选择文本并复制
# 启用这种模式后，你可以通过鼠标选择来复制文本
bind -T copy-mode-vi MouseDragEnd1Pane send-keys -X copy-pipe-and-cancel "xclip -in -selection clipboard"
```

部分终端可能直接鼠标滚轮即可在pane中翻页，部分需要配合**shift键**才能翻页。