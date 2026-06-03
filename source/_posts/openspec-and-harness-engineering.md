---
title: AI编程三剑客：OpenSpec + Harness Engineering 详解
date: 2026-06-04 00:30:00
tags: AI, programming, OpenSpec, Harness
categories: Posts
excerpt: 2026年AI编程领域最火的规范驱动开发与Harness工程化方法论，详解OpenSpec和Harness的核心概念、工作流与实战用法
---

> 2026年，AI编程领域的竞争焦点已经从"谁的模型更强"转向"谁的基础设施更完善"。**OpenSpec、Harness Engineering 和 Superpowers** 被称为 AI 工程化开发的三层拼图。本文重点详解前两个。

---

## 一、为什么需要它们？

用 AI 写代码时，你是否遇到过这些问题：

- **AI 经常跑偏** — 你说要 A，它给你写了 B
- **结果不可预测** — 同样的需求，每次生成结果不同
- **质量参差不齐** — 缺乏工程纪律，代码质量无法保证
- **多 Agent 协作混乱** — 分工不清，互相冲突

**OpenSpec 解决"方向"问题，Harness 解决"纪律"问题。**

---

## 二、OpenSpec — 定方向

### 2.1 是什么

**OpenSpec** 是由 Fission-AI 团队开源的 **规范驱动开发（Spec-Driven Development，SDD）框架**。

核心理念：**在写代码之前，先让人和 AI 就"要做什么"达成共识。**

- **GitHub：** [Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec)
- **License：** MIT
- **特点：** 轻量、纯本地（Markdown 文件）、不需要 API Key、支持 28+ AI 工具

### 2.2 核心概念

```
┌─────────────────────────────────────────────────┐
│  Specs（规范）                                    │
│  当前系统行为的"真相源"                              │
│  用 Given/When/Then 格式描述需求                     │
└─────────────────────────────────────────────────┘
          ▲ 归档时合并
          │
┌─────────────────────────────────────────────────┐
│  Changes（变更）                                   │
│  一个变更提案 = proposal + specs + design + tasks  │
│  Delta Specs 只写增/改/删部分                       │
└─────────────────────────────────────────────────┘
```

| 概念 | 说明 |
|------|------|
| **Specs** | 当前系统行为的真相源，按领域组织（`auth/`、`payments/` 等） |
| **Changes** | 一个变更提案，包含完整的规划工件 |
| **Delta Specs** | 增量规范，只写 ADDED / MODIFIED / REMOVED 部分 |
| **Archive** | 变更完成后归档，Delta 合并进主规范 |

### 2.3 安装与初始化

```bash
# 安装（需要 Node.js ≥ 20.19.0）
npm install -g @fission-ai/openspec@latest

# 验证
openspec --version

# 在项目中初始化
cd your-project
openspec init
# 或指定工具
openspec init --tools claude,cursor
```

初始化后生成如下目录结构：

```
openspec/
├── specs/              # 真相源（当前系统行为）
│   └── <domain>/
│       └── spec.md
├── changes/            # 变更提案
│   └── <change-name>/
│       ├── proposal.md    # 为什么做、做什么
│       ├── design.md      # 怎么做
│       ├── tasks.md       # 任务清单
│       └── specs/         # Delta 规范
└── config.yaml         # 项目配置（可选）
```

### 2.4 工作流：三步搞定

在 AI 助手（Claude Code / Cursor / Copilot 等）中使用斜杠命令：

```text
/opsx:propose 添加用户注册功能    → 自动生成提案+规范+设计+任务清单
/opsx:apply                     → AI 按任务清单逐项实现代码
/opsx:archive                   → 归档变更，Delta 合并进主规范
```

**实战示例：添加暗黑模式**

```text
你：/opsx:propose add-dark-mode

AI：✓ 创建 openspec/changes/add-dark-mode/
    ✓ proposal.md  — 为什么要做、影响范围
    ✓ specs/       — 需求和场景描述
    ✓ design.md    — 技术方案（CSS 变量 + Context）
    ✓ tasks.md     — 实现清单

你：/opsx:apply

AI：✓ 1.1 添加 ThemeContext Provider
    ✓ 1.2 创建 Toggle 组件
    ✓ 2.1 定义 CSS 变量
    ✓ 2.2 接入 localStorage 持久化

你：/opsx:archive

AI：✓ 归档到 changes/archive/2026-06-04-add-dark-mode/
    ✓ 主规范已更新，可以开始下一个需求
```

### 2.5 更多命令

| 命令 | 用途 |
|------|------|
| `/opsx:explore` | 探索想法，不提交变更 |
| `/opsx:verify` | 验证实现是否符合规范 |
| `/opsx:new` | 手动创建变更（逐步推进） |
| `/opsx:ff` | 快进：一次性生成所有规划工件 |
| `/opsx:bulk-archive` | 批量归档多个变更 |
| `/opsx:onboard` | 交互式教程 |

CLI 端常用命令：

```bash
openspec list              # 查看所有变更
openspec view <change>     # 查看变更详情
openspec validate --all    # 验证所有规范
openspec archive <change>  # 归档变更
openspec status            # 查看当前状态
```

### 2.6 兼容的 AI 工具（28+）

Claude Code、Cursor、Windsurf、Codex、GitHub Copilot、Cline、Gemini CLI、Kimi、Kiro、Continue、Trae 等。

---

## 三、Harness Engineering — 定纪律

### 3.1 是什么

**Harness Engineering** 是由 Martin Fowler、OpenAI、Addy Osmani 等人推广的 AI Agent 工程化方法论。

核心公式：

> **Agent = Model + Harness**

**Harness（马具）** 就是套在 AI 模型外面的一切基础设施 — 提示词、工具、沙箱、钩子、记忆、子代理、反馈回路。

**一句话总结：** AI Agent 是一匹潜力无限的野马，Harness 就是那套将它驯化为千里马的驾驭体系。

### 3.2 为什么 Harness 比模型更重要

> *"一个好模型配上差的 Harness，不如差模型配上好的 Harness。"*
> — Addy Osmani（Google Chrome 团队）

有团队仅靠优化 Harness（不换模型），就把编码 Agent 从 **Top 30 提升到 Top 5**。

**我们和 AI 之间的差距，更多是 Harness 差距，不是模型差距。**

### 3.3 核心框架：Guide 与 Sensor

Martin Fowler 提出了两个核心概念：

| 概念 | 方向 | 说明 |
|------|------|------|
| **Guides（前馈）** | 行动前 | System Prompt、AGENTS.md、Skills、规则文件 |
| **Sensors（反馈）** | 行动后 | Linter、类型检查、代码审查 Agent、测试 |

```
  Guides（前馈）
  ┌─────────────┐
  │ AGENTS.md    │ ──→ 告诉 Agent 该怎么做
  │ Skills       │
  │ Rules        │
  └──────┬──────┘
         ▼
    Agent 行动
         │
         ▼
  ┌─────────────┐
  │ Sensors     │ ──→ 检查 Agent 做对了没
  │ Linter      │
  │ Tests       │
  │ Code Review │
  └─────────────┘
```

**优先用计算型传感器**（Linter、类型检查，快且确定），再叠加推理型传感器（AI 审查，慢但有语义理解）。

### 3.4 棘轮原则（Ratchet Principle）

**这是 Harness Engineering 最核心的实践：**

> Agent 犯的每个错误 → 变成永久规则。

```
Agent 注释掉了测试
    ↓
AGENTS.md 新增规则："禁止注释测试代码"
    ↓
pre-commit hook 添加 grep 检查
    ↓
代码审查 Agent 标记同类行为
    ↓
下次不会再犯 → 棘轮又转了一格
```

**每条 AGENTS.md 中的规则，都要能追溯到一次真实失败。** 不凭空想象规则，只从实际错误中提炼。

### 3.5 Harness 的核心组件

| 组件 | 作用 | 示例 |
|------|------|------|
| **System Prompt / AGENTS.md** | 注入规则和上下文 | Claude Code 的 CLAUDE.md |
| **Tools / MCP** | 让 Agent 执行真实操作 | 文件读写、终端、浏览器 |
| **Hooks** | 拦截+自动纠正 | 阻止 `rm -rf`、编辑后自动格式化 |
| **Sensors** | 事后检查 | Lint、类型检查、测试 |
| **Subagents** | 生成与评估分离 | 一个写代码，一个审查 |
| **Memory** | 跨会话知识持久化 | 避免重复犯错 |
| **Sandbox** | 隔离执行环境 | Docker 容器 |

### 3.6 热门开源 Harness 项目

#### DeerFlow（字节跳动）⭐ 49k+

基于 LangGraph + LangChain 的超级 Agent Harness：

- **子代理系统** — Lead Agent 并行分发任务给专业子代理
- **沙箱** — 本地 / Docker / K8s 隔离执行
- **长期记忆** — 跨会话用户画像和知识库
- **多 IM 渠道** — Telegram、Slack、飞书、微信、钉钉
- **可观测性** — LangSmith + Langfuse 链路追踪

#### oh-my-pi ⭐ 8.9k+

Rust 编写的终端优先编码 Agent Harness：

- 40+ Provider，32 内置工具
- LSP 实时诊断 + DAP 真实调试器
- Hashline 编辑（按内容哈希定位，不怕文件变动）
- 双内核执行（Python + Bun）

### 3.7 实践 Harness Engineering 的步骤

```
1. 写 AGENTS.md         ← 从第一条规则开始
2. 加计算型传感器        ← Lint、类型检查（快、确定性）
3. 加 Hooks              ← 拦截破坏性操作、编辑后自动检查
4. 叠加推理型传感器      ← AI 代码审查
5. 建设记忆系统          ← 跨会话积累知识
6. 引入子代理            ← 生成和评估分离
7. 持续迭代              ← 每次错误 → 新规则 → 棘轮转动
```

---

## 四、三剑客组合：规范 → 纪律 → 流程

| 层级 | 工具 | 职责 | 一句话 |
|------|------|------|--------|
| **规范层** | OpenSpec | 定方向 | 把模糊需求变成严格规范 |
| **纪律层** | Harness Engineering | 定纪律 | 用工程化手段约束 Agent 行为 |
| **执行层** | Superpowers | 定流程 | 统一开发流程和工具链 |

**推荐落地路径：**

```
先上 OpenSpec（稳定需求规范）
    ↓
再加 Harness 规则（约束 Agent 行为）
    ↓
最后上多 Agent 协作（Superpowers）
```

> 不要一上来就搞多 Agent 协作，先把单个 Agent 的规范和纪律搞定。

---

## 五、关键金句

> *"If you're not the model, you're the harness."*
> — Viv Trivedy

> *"It's not a model problem. It's a configuration problem."*
> — HumanLayer

> *"A good harness should not aim to fully eliminate human input, but to direct it to where our input is most important."*
> — Birgitta Böckeler / Martin Fowler

---

## 六、参考资源

- **OpenSpec GitHub：** [Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec)
- **DeerFlow GitHub：** [bytedance/deer-flow](https://github.com/bytedance/deer-flow)
- **oh-my-pi GitHub：** [can1357/oh-my-pi](https://github.com/can1357/oh-my-pi)
- **Martin Fowler：** [Guides and Sensors](https://martinfowler.com/)
- **awesome-harness-engineering：** [ai-boost/awesome-harness-engineering](https://github.com/ai-boost/awesome-harness-engineering)

---

> 2026 年的 AI 编程，拼的不是谁的 Prompt 写得好，而是谁的基础设施建得牢。OpenSpec 让你"想清楚再动手"，Harness 让你"犯了错就记住"。这两者结合，才是 AI 编程的正确打开方式。
