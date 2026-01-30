---
title: Moltbot(Clawdbot) 在 macOS 系统中的部署指南
date: 2026-01-29 16:26:06
tags: [tools, LLM]
categories: Posts
excerpt: 本文档详细介绍了如何在 macOS 系统中部署 Clawdbot、安装技能(Skills)以及保持后台运行的方法。
---

## 1. 系统要求

- macOS 10.15 (Catalina) 或更高版本
- Node.js 18.x 或更高版本
- npm 包管理器
- 稳定的互联网连接

## 2. 安装 Clawdbot

### 2.1 安装 Node.js

首先确保您的系统已安装 Node.js：

```bash
# 检查是否已安装 Node.js
node --version
npm --version

# 如果未安装，可以通过以下方式之一安装：
# 方式1：使用 Homebrew
brew install node

# 方式2：从官网下载安装包
# https://nodejs.org/
```

### 2.2 全局安装 Clawdbot

```bash
# 全局安装 Clawdbot
npm install -g clawdbot
## 或者
curl -fsSL https://molt.bot/install.sh | bash

# 验证安装
clawdbot --version
```

## 3. 配置 Clawdbot

### 3.1 运行配置向导

```bash
# 运行交互式配置向导
clawdbot configure
```

### 3.2 配置通道（Channels）

根据您的需要配置通信渠道，如 Telegram、WhatsApp 等：

```bash
# 配置 Telegram 机器人
clawdbot channels login --channel telegram

# 配置 WhatsApp
clawdbot channels login --channel whatsapp
```

## 4. 安装和管理 Skills

[MoltHub](https://clawdhub.com)

### 4.1 使用 ClawdHub 搜索和安装 Skills

```bash
# 安装
npm i -g clawdhub

# 搜索可用的 Skills
npx clawdhub search [关键词]

# 安装特定 Skill
npx clawdhub install [skill-name]

# 查看已安装的 Skills
npx clawdhub list

# 更新所有 Skills
npx clawdhub update
```

### 4.2 手动安装 Skills

您也可以从 GitHub 或其他来源手动安装 Skills：

```bash
# 进入工作目录
cd ~/clawd

# 创建 Skills 目录（如果不存在）
mkdir -p skills

# 将 Skill 文件夹复制到 skills 目录
cp -r /path/to/skill ./skills/
```

### 4.3 配置 API 密钥

对于需要 API 密钥的 Skills，在配置文件中设置：

```bash
# 编辑配置文件
nano ~/.clawdbot/clawdbot.json

# 在配置文件中添加 API 密钥
{
  ...
  skills: {
    entries: {
      "skill-name": {
        enabled: true,
        # ENV_API_KEY 对应skills需要的环境变量名
        apiKey: "ENV_API_KEY",
        env: {
          ENV_API_KEY: "YOUR-API-KEY"
        },
      },
    }
  }
}
```

### 4.4 验证 Skills 安装

```bash
# 检查所有 Skills 的状态
clawdbot skills check

# 查看特定 Skill 信息
clawdbot skills info [skill-name]
```

## 5. 启动和测试 Clawdbot

### 5.1 启动 Gateway

```bash
# 启动clawdbot
clawdbot onboard

# 启动 Clawdbot Gateway
clawdbot gateway
```

### 5.2 测试功能

启动后，通过您配置的通信渠道（如 Telegram 或 WhatsApp）与 Clawdbot 交流，确认一切正常工作。

## 6. 保持后台运行

### 6.1 方法一：使用 PM2（推荐）

#### 安装 PM2

```bash
# 全局安装 PM2
npm install -g pm2

# 启动 Clawdbot 并命名进程
pm2 start clawdbot --name "clawdbot" -- gateway

# 设置开机自启
pm2 startup
pm2 save

# 查看进程状态
pm2 status

# 查看日志
pm2 logs clawdbot
```

#### PM2 常用命令

```bash
# 重启进程
pm2 restart clawdbot

# 停止进程
pm2 stop clawdbot

# 删除进程
pm2 delete clawdbot

# 查看内存和 CPU 使用情况
pm2 monit
```

### 6.2 方法二：使用 Launchd（macOS 原生）

创建一个 Launchd 配置文件：

```bash
# 创建 plist 文件
nano ~/Library/LaunchAgents/com.user.clawdbot.plist
```

将以下内容粘贴到文件中（替换路径为您的实际路径）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.clawdbot</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/clawdbot</string>
        <string>gateway</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/[YOUR_NAME]/clawd</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/[YOUR_NAME]/clawdbot.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/[YOUR_NAME]/clawdbot_error.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
    </dict>
</dict>
</plist>
```

加载服务：

```bash
# 加载服务
launchctl load ~/Library/LaunchAgents/com.user.clawdbot.plist

# 启动服务
launchctl start com.user.clawdbot

# 查看服务状态
launchctl list | grep clawdbot

# 停止服务
launchctl stop com.user.clawdbot

# 卸载服务
launchctl unload ~/Library/LaunchAgents/com.user.clawdbot.plist
```

### 6.3 方法三：使用 nohup

```bash
# 在后台启动 Clawdbot
nohup clawdbot gateway > ~/clawdbot.log 2>&1 &

# 获取进程 ID
echo $! > ~/clawdbot.pid

# 检查进程是否在运行
ps aux | grep clawdbot

# 停止进程
kill $(cat ~/clawdbot.pid)
```

### 6.4 方法四：使用 tmux

```bash
# 创建并启动 tmux 会话
tmux new-session -d -s clawdbot 'clawdbot gateway'

# 重新连接到会话
tmux attach -t clawdbot

# 分离会话（保持运行）
# 在 tmux 会话中按 Ctrl+b 然后按 d

# 列出所有会话
tmux ls

# 停止会话
tmux kill-session -t clawdbot
```

## 7. 监控和维护

### 7.1 日志管理

无论使用哪种后台运行方法，都建议定期检查日志：

```bash
# 如果使用 PM2
pm2 logs clawdbot --lines 100

# 如果使用 Launchd，查看指定的日志文件
tail -f ~/clawdbot.log
```

### 7.2 更新 Clawdbot

```bash
# 更新 Clawdbot
npm update -g clawdbot

# 重启服务（如果是 PM2 管理）
pm2 restart clawdbot
```

### 7.3 备份配置

定期备份您的配置文件：

```bash
# 备份配置
cp -r ~/.clawdbot ~/clawdbot_backup_$(date +%Y%m%d)
```

## 8. 故障排除

### 8.1 常见问题

1. **权限问题**：确保运行 Clawdbot 的用户有足够的权限
2. **端口占用**：默认端口 18789，如果被占用，使用 `--port` 参数指定其他端口
3. **API 密钥**：确保所有需要的 API 密钥都已正确配置
4. **依赖问题**：有的时候npm安装缺少依赖，需要手动补齐，在安装`clawdhub`时没有安装依赖`undici`，`cd /opt/homebrew/lib/node_modules/clawdhub &&
npm install undici`

### 8.2 调试技巧

```bash
# 运行诊断
clawdbot doctor

# 检查健康状态
clawdbot health

# 查看状态
clawdbot status
```

## 9. 性能优化

- 定期清理日志文件
- 监控内存使用情况
- 根据需要调整模型设置
- 只启用必要的 Skills 以减少资源消耗

## 其他

### Telegram 机器人配置

1. 搜索BotFather
2. 发送`/newbot`
3. 会让你输入机器人的名字，随便定义，比如：`clawdbot`
4. 输入机器人用户名，比如：`xxxbot`，这里必须以bot结尾，用户名是唯一id，可能会因为占用需要重新输入
5. 创建完后会收到HTTP API的密钥
6. 发送`/setprivacy`，然后发送你刚才创建的机器人的用户名id
7. 此时输入框会显示选项，选择`Disable`
8. 创建完后可以在搜索通过`@xxxbot`找到对应机器人添加
9. 第一次需要主动跟机器人对话，会返回一个 `Pairing code`
10. 运行`clawdbot pairing approve telegram <code>`


