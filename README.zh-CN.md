[English](./README.md) | **汉语**

# AlfredContext

**为 AI Coding Agent 提供的 Git 版本化项目记忆** —— 编译出低 Token 上下文，让 Codex、Claude Code 等工具接着上一个工具继续干活。

`alfredc` · `.alfredc/` · [协议 v0.1](./spec/protocol.md)

---

## 要解决的问题

额度限制迫使你在不同的 AI Coding Agent 之间来回切换。今天用 Codex，明天用 Claude Code，后天换成 OpenCode。

每个新 Agent 都是两眼一抹黑。它不知道上一个工具做过什么。于是你要么浪费大量 token 重新教它认识整个项目，要么眼睁睁看着它重犯你两天前刚纠正过的错误。

真正丢失的是**细节**。不是「登录页已经做完了」，而是「用户因为移动端交互问题否决了 Modal 方案，而且这个决定依然生效」。

名字取自那位管家：蝙蝠侠换了一代又一代，Alfred 始终是同一个。

## AlfredContext 是什么

它不是又一个 AI 记忆工具，而是一个**项目连续性与审计层**，垫在你正在使用的任何 Coding Agent 之下。

它把开发历史记录成一份 Git 版本化的账本，再从中编译出新 Agent 真正需要的那一小块高信号上下文。

三个词承载了它的全部重量：**连续性（Continuity）**、**可追溯（Provenance）**、**上下文编译（Context Compilation）**。

## 工作原理

事件溯源（Event Sourcing）+ 物化视图（Materialized Context）。

```
        账本 —— 只追加，唯一真相来源
        .alfredc/history/2026-09-15.jsonl
                      │
                      │  编译 · 随时可重建
                      ▼
        物化视图 —— 给大模型看的内容
        brief.md · constraints.md · current.md
```

每条事件都带有**权威等级**，并且可以**废止（supersede）**更早的事件：

| 事件类型 | 权威 |
| --- | --- |
| `user_correction`（用户纠正） | 最高 |
| `user_requirement`（用户要求） | ▲ |
| `accepted_decision`（已确认决策） | │ |
| `project_fact`（项目事实） | │ |
| `verification`（验证结果） | │ |
| `agent_hypothesis`（Agent 推断） | 最低 |

**这正是解决问题的关键。** 当你否决一个方案时，一条 `user_correction` 会废止它。此后每一个 Agent 都能看到这条生效的约束**以及它存在的原因**——而不是一份悄悄把它弄丢的摘要。

因为账本是真相、视图是派生物，所以反复摘要压缩也不会丢失信息。新 Agent 拿到的是约 2,000 token 的精华，而不是重读 500 个文件加 8 万 token 的聊天记录。

## 零模型配置

你**不需要**为 AlfredContext 配置任何模型。提炼复用你正在使用的那个 Coding Agent——不需要 API key，不需要额外账号，不产生额外账单。

## 项目状态

**协议先行。目前还没有代码。**

| | |
| --- | --- |
| ✅ | **协议 v0.1 已定稿** —— [`spec/protocol.md`](./spec/protocol.md) |
| ✅ | **已自举** —— 本仓库自己的开发历史就存放在 [`.alfredc/`](./.alfredc/) 中 |
| ⬜ | **`alfredc` CLI 尚未实现** —— Go 骨架还未动工 |

现在交付的就是协议本身。如果你想在格式冻结之前影响它，现在是最好的时候。

## 仓库结构

| 路径 | 说明 |
| --- | --- |
| [`spec/protocol.md`](./spec/protocol.md) | **公开规格。** 规范性的——实现以此为准。 |
| [`design/`](./design/) | 内部研发文档。**非**规范性；可能包含未经验证的调研。 |
| [`.alfredc/`](./.alfredc/) | 本项目自己的记忆——AlfredContext 在自举自己。 |
| [`friction-log.md`](./friction-log.md) | 协议在实际使用中每一处别扭的地方。同时是需求清单。 |

## 计划支持的 Agent

v1 计划支持：

| 工具 | 仓库指令 | 状态 |
| --- | --- | --- |
| Claude Code | `CLAUDE.md` → `AGENTS.md` | ⬜ 计划中 |
| Codex | `AGENTS.md` | ⬜ 计划中 |
| OpenCode | `AGENTS.md` | ⬜ 计划中 |
| ZCode | `AGENTS.md` | ⬜ 计划中 |
| Qoder | `AGENTS.md` / `.qoder/rules` | ⬜ 计划中 |

> ⚠️ **这些集成是计划，不是已验证的事实。** 各工具的 Hook 能力与 headless 入口**尚未逐条比对官方文档**。记录于 [`.alfredc/current.md`](./.alfredc/current.md) 的 `Q-001`。

## 非目标

- **不是一个模型。** 它从不调用任何 LLM API，也不持有任何凭据。
- **不做 MCP Server**（v1）。
- **不做向量数据库**或语义检索。
- **不做聊天记录全文存档。**
- **不替代**任何 Coding Agent。

## 文档

- [协议 v0.1](./spec/protocol.md) —— 规范性规格
- [当前状态](./.alfredc/current.md) —— 已完成、下一步、待决问题
- [摩擦日志](./friction-log.md) —— 设计在哪里别扭，以及因此改了什么
- [设计笔记](./design/) —— 背景与早期探索

## 参与贡献

现在还没有代码可供贡献——但这也正是对设计提出异议的最佳时机。

协议在 [`spec/protocol.md`](./spec/protocol.md)。如果其中有错误、歧义或缺失，**此刻提一个 issue 比提一个 PR 更有价值。**

## 许可证

[Apache-2.0](./LICENSE) —— 版权所有 © 2026 ShawnZhang31。

这是刻意的选择。明确的专利授权与宽松条款，让企业可以放心采用 AlfredContext，并用自己的 Adapter 实现该协议——而这正是公开一份规格的全部意义。
