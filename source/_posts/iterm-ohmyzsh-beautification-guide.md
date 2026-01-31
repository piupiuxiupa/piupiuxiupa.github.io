---
title: iTerm2 + Oh My Zsh 美化配置完整指南
date: 2026-01-31 22:47:51
tags: tools
categories: Posts
excerpt: 本文档将详细介绍如何在 macOS 系统中配置 iTerm2 和 Oh My Zsh，实现终端的全面美化。
---

## 1. 系统要求和前置安装

### 1.1 必需组件

```bash
# 安装 Homebrew（如果没有）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 安装 Oh My Zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# 安装 zsh 插件管理器 zinit (推荐) 或 zplug
# 或者使用内置的 git 插件管理
```

### 1.2 安装 Nerd Fonts 字体

```bash
# 安装 Nerd Fonts (推荐 Meslo LG M 或 JetBrains Mono)
brew tap homebrew/cask-fonts
brew install --cask font-jetbrains-mono-nerd-font
brew install --cask font-hack-nerd-font
brew install --cask font-meslo-lg-nerd-font
```

## 2. Oh My Zsh 配置

### 2.1 编辑 `.zshrc` 配置文件

```bash
# 备份原配置
cp ~/.zshrc ~/.zshrc.backup

# 编辑配置文件
nano ~/.zshrc
```

### 2.2 基础主题配置

```bash
# 在 ~/.zshrc 中配置主题
ZSH_THEME="powerlevel10k/powerlevel10k"
# 或者
# ZSH_THEME="spaceship/spaceship"
```

### 2.3 插件配置

推荐添加以下插件以增强功能：

```bash
plugins=(
  git 
  web-search
  extract
  z
  zsh-syntax-highlighting  # 语法高亮
  zsh-autosuggestions    # 自动补全
  colored-man-pages      # 彩色手册页
  sudo                   # 快速添加 sudo
  copyfile               # 快速复制文件内容到剪贴板
  copypath               # 快速复制路径到剪贴板
  docker                 # Docker 命令补全
  docker-compose         # Docker Compose 命令补全
  yarn                   # Yarn 命令补全
  node                   # Node.js 相关
  npm                    # npm 命令补全
  brew                   # Homebrew 命令补全
  osx                    # macOS 特定命令
)
```

### 2.4 安装语法高亮和自动补全插件

```bash
# 安装 zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

# 安装 zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-autosuggestions.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

### 2.5 特定主题配置

#### Powerlevel10k 配置

运行配置向导：

```bash
p10k configure
```

#### Spaceship 主题配置

```bash
# spaceship 主题设置
SPACESHIP_USER_SHOW="always"
SPACESHIP_USER_COLOR="212"
SPACESHIP_DIR_TRUNC_REPO=false
```

### 2.6 自定义别名

根据您的 `.zshrc`，您已设置以下别名：

```bash
alias proxy='export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890'
alias unproxy='unset https_proxy http_proxy all_proxy'
alias ll='ls -lh'
```

推荐添加以下实用别名：

```bash
# 导航
alias ..="cd .."
alias ...="cd ../.."
alias ....="cd ../../.."
alias ~="cd ~"
alias -- -="cd -"

# 列出文件
alias ls='ls -GFh'
alias la='ls -A'
alias ll='ls -lh'
alias lla='ll -A'

# Git 别名
alias gst='git status'
alias gb='git branch'
alias gc='git commit'
alias ga='git add'
alias gco='git checkout'
alias gd='git diff'
alias gl='git pull'
alias gp='git push'

# 系统管理
alias df='df -h'
alias du='du -h'
alias mkdir='mkdir -pv'

# 网络
alias myip='curl ip.appspot.com'
alias flushdns='sudo dscacheutil -flushcache'
```

## 3. iTerm2 配置

### 3.1 安装和基础配置

1. 从官网下载并安装 iTerm2
2. 打开 iTerm2，进入 Preferences (Cmd+,)

### 3.2 字体配置

1. 进入 `Preferences > Profiles > Text`
2. 设置 Regular Font 和 Non-ASCII Font 为刚才安装的 Nerd Font
   - 推荐: `MesloLGS NF` 或 `JetBrainsMono Nerd Font`
   - 字体大小: 14-16 (根据屏幕分辨率调整)

> 下载`powerline`字体 https://github.com/powerline/fonts.git

### 3.3 颜色主题配置

1. 下载 [iTerm2-Color-Schemes](https://github.com/mbadolato/iTerm2-Color-Schemes)
2. 进入 `Preferences > Profiles > Colors`
3. 点击 `Color Presets... > Import`，导入喜欢的主题
4. 推荐主题：
   - Dracula
   - One Dark
   - Solarized Dark
   - Gruvbox Dark

### 3.4 窗口和外观配置

1. 进入 `Preferences > Profiles > Window`
   - Style: Full Screen 或 Top of Screen
   - Transparency（设置透明度）: 5-15% (可选)
   - Blur（设置模糊效果）: On (可选)

2. 进入 `Preferences > Appearance`
   - Theme: Minimal
   - Show tab bar even when there is only one tab: 取消勾选
   - Dimming amount: 0.1 (可选)

3. 进入 `Preferences > Terminal`
   - 取消勾选 `Show mark indicators` 去掉左侧三角箭头

### 3.5 键盘快捷键配置

1. 进入 `Preferences > Keys > Key Bindings`
2. 添加以下有用的快捷键:
   - Cmd+Shift+T: 全屏下显示tab
   - Cmd+T: 新建标签页
   - Cmd+左右箭头: 切换标签页
   - Cmd+[, Cmd+]: 切换同标签页下的不同panel
   - Cmd+D: 垂直分屏
   - Cmd+Shift+D: 水平分屏
