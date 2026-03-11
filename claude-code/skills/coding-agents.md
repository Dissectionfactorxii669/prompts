---
name: coding-agents
description: "Codex CLI 和 Gemini CLI 协作开发规范。当需要调度外部 coding agent 写代码、做前端、review 代码时使用。触发词：用 codex、用 gemini、让 agent 写、前端开发、后端开发、协作开发。"
version: 1.0.0
---

# Coding Agent 协作手册

Claude Code 作为调度者，调度 Codex CLI（后端/业务）和 Gemini CLI（前端/UI）完成开发任务。

## 角色分工

| 角色 | 工具 | 擅长 | 缺陷 |
|------|------|------|------|
| **调度者** | Claude Code | 方案设计、上网查资料、深度推理、review、MCP 工具 | 批量写码较慢 |
| **后端执行者** | Codex CLI | Go/Rust/Python 业务逻辑、系统编程 | 无网络、sandbox 受限 |
| **前端执行者** | Gemini CLI | React/Vue/CSS/HTML、UI 组件、样式调优 | 无网络 |

## Codex CLI

### 配置

- Binary: `codex` (npm global)
- Config: `~/.codex/config.toml`
- Sessions: `~/.codex/sessions/`

### 常用命令

```bash
# 非交互执行（全自动，sandbox 内可读写工作区）
codex exec --full-auto -C /path/to/repo "任务描述"

# 恢复 session 继续（非交互）
codex exec resume --last "补充信息或继续指令"
codex exec resume <session-id> "补充信息"

# Review 当前未提交改动
codex review --uncommitted

# 交互模式（用户在终端操作）
codex -C /path/to/repo "任务描述"
codex resume --last  # 恢复交互 session
```

### 注意事项

- sandbox 内无法 `go get` / `npm install`（网络受限），依赖需要调度者在外部补
- `codex exec` 完成后会输出 `tokens used` 标记
- 可能会擅自修改 AGENTS.md / README，review 时注意还原不必要的改动

## Gemini CLI

### 配置

- Binary: `gemini` (npm global)
- Config: `~/.gemini/settings.json`
- Sessions: `~/.gemini/history/`

### 常用命令

```bash
# 非交互执行（yolo = 自动批准所有操作）
gemini --prompt "任务描述" --yolo

# JSON 输出（获取 session_id）
gemini --prompt "任务描述" --yolo --output-format json

# 恢复 session
gemini --resume latest
gemini --resume <index>

# 列出 sessions
gemini --list-sessions

# 交互模式
gemini "任务描述"
gemini --resume latest  # 恢复
```

### 前端开发要点

- 擅长 React 组件、Tailwind CSS、响应式布局
- 可以直接读写项目文件、运行 dev server
- 给足设计上下文：截图、现有组件风格、色板

## 协作工作流

```
Claude Code (调度者)
 |
 |-- 1. 分析需求、设计方案
 |-- 2. 写 prompt（含必要上下文、文件路径、约束）
 |-- 3. 启动 Agent
 |     |-- 后端: codex exec --full-auto -C <repo> "prompt"
 |     |-- 前端: gemini --prompt "prompt" --yolo
 |-- 4. 监控输出
 |     |-- Agent 卡住 → 我查资料 → session resume + 知识注入
 |     |-- Agent 报错 → 我诊断 → session resume + 修复指令
 |     |-- Agent 完成 → 进入 review
 |-- 5. Review: git diff → build/test → 反馈
 |     |-- 小问题 → session resume + 修复指令
 |     |-- 根本问题 → 新 task + 更好的 prompt
 |-- 6. 通过 → commit + push
```

## Session Continue vs 新 Task

### 用 Session Continue 的场景

| 场景 | 操作 |
|------|------|
| 缺外部知识被卡住 | resume + 注入查到的 API 文档/签名 |
| 编译/构建错误 | resume + 错误信息和修复方向 |
| 部分完成需继续 | resume + "接下来做 X" |
| Review 后小修 | resume + "修这几个点: 1. 2. 3." |

核心判断：**Agent 的方向是对的，只是缺一块拼图。** Continue 保留了它对任务的理解。

### 用新 Task 的场景

| 场景 | 操作 |
|------|------|
| 方向根本错了 | 新 prompt，修正架构/约束 |
| 完全独立的新工作 | 新 task，不需要旧上下文 |
| Session 上下文太大 | 带总结重开，减少 token 开销 |
| 并行任务 | 多个独立 exec 同时跑 |

核心判断：**上下文是资产还是负债？** 资产 → continue，负债 → 新 task。

## Prompt 编写原则

1. **上下文充分** — 列出要读的关键文件路径、现有 API/类型签名
2. **约束明确** — "不要改 X"、"不要加测试"、"只用 Y 库"
3. **目标可验证** — "编译通过"、"输出包含 Z"
4. **一个 prompt 一个任务** — 不要把多个不相关的改动混在一起

## 典型 Prompt 模板

```
## Task: <一句话描述>

### Context
<项目背景、架构简述>

### What to implement
<具体步骤，按文件分节>

### Key files to read first
- `path/to/file.go` — 描述
- `path/to/other.ts` — 描述

### Design constraints
- <约束 1>
- <约束 2>

### Do NOT
- <不要做的事>
```

## 任务分配决策

```
这个任务是什么类型？
|-- Go / Rust / Python 后端逻辑 → Codex
|-- React / Vue / CSS 前端 → Gemini
|-- 方案设计 / 调研 / Review → Claude Code 自己做
|-- 需要上网查 → Claude Code 查完给 Agent
|-- 跨前后端 → 拆成两个 task 分别给
```
