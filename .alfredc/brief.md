---
id: brief
updated: 2026-09-15T00:00:00Z
---

# Project Brief

> 本文件是**半手工**文件（协议 §3.2 唯一例外）：表达项目长期愿景，不随单次会话变化。

## What

AlfredContext 是一个**独立于 Coding Agent 的项目连续性协议与审计运行时**。

它把开发过程中产生的需求、纠正、决策、尝试、验证、工作状态沉淀为 **Git 可版本化的项目记忆**，并按任务**动态生成低 Token 上下文**，使不同 Coding Agent 能够无缝接力。

CLI 命令为 `alfredc`，项目记忆目录为 `.alfredc/`。

## Why

使用 AI Coding 工具时，额度限制迫使开发者在 Codex、Claude Code、OpenCode、ZCode、Qoder 之间不停切换。切换后新工具**不知道前任做过什么**，只能：

- 浪费大量 token 重新熟悉整个项目；
- 仍然不了解此前对话的细节与设计约束；
- **重犯前任已被用户纠正过的错误。**

已有的 Memory 类工具只解决其中一部分。AlfredContext 的差异在于它不是"又一个 AI 记忆"，而是**面向多工具的项目连续性与开发审计层**——不替代任何 Coding Agent，而是让它们共享同一个 Alfred。

## 三个护城河词

- **Continuity**（连续性）：换工具/换模型，不重新熟悉项目。
- **Provenance**（可追溯）：任何一条约束都能追到「谁、何时、为什么」。
- **Context Compilation**（上下文编译）：不是塞入全部历史，而是按需编译出精炼上下文。

## Tech Stack

**Go**（协议 §12.1）。理由：Hook 高频调用要求冷启动 ~2-5ms、纯 Go SQLite 免 cgo 实现零依赖静态单二进制、五工具 Hook 多为 shell 命令天然语言无关。

## Non-goals

- ❌ 不自带、不托管、不代理任何模型——AlfredContext 永不是一个"AI 产品"，它只调度用户已有的 Coding Agent。
- ❌ 不做 MCP Server（v1）。
- ❌ 不做 Vector DB / 语义检索。
- ❌ 不做聊天记录全文存档。
- ❌ 不替代任何 Coding Agent 的调度/执行能力。

## 单一事实来源

`spec/protocol.md` 是核心协议的单一事实来源。`design/preview.md` 是最初的想法探讨（背景，非规范）。
