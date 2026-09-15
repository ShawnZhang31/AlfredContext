<!-- generated: do not edit; use `alfredc record` -->
# Requirements

> 所有 `user_requirement` 按状态分组，供需求审计。来源：`history/*.jsonl`。

## Active

| ID | Requirement | Priority | Scope | 日期 |
| --- | --- | --- | --- | --- |
| REQ-001 | 跨 AICoding 工具的项目连续性与开发审计能力：新工具不必重新熟悉项目，且不重犯已被用户纠正的错误 | critical | `**` | 09-14 |
| REQ-002 | 项目名 **AlfredContext**，CLI 命令 **`alfredc`** | high | `**` | 09-14 |
| REQ-003 | 至少支持 Codex、Claude Code、OpenCode、ZCode、Qoder 五种工具 | high | `**` | 09-14 |
| REQ-004 | 产出内容可提交到代码仓库，供团队协作使用 | high | `**` | 09-14 |
| REQ-005 | README 英文为主 + `README.zh-CN.md`，顶部各放语言选择链接 | normal | `README*.md` | 09-14 |

## Superseded

（无）

## 关联约束

REQ-001 是项目的**根本需求**，其余需求与全部已接受决策都应可回溯到它。特别是：

- **COR-002**（不得要求用户配置模型）— 服务于 REQ-001 的"轻量"前提：若要额外申请模型账号，工具就不会被真正用起来。
- **DEC-003 / DEC-012**（机制 B + 用现有 Coding Agent 提取）— 服务于 REQ-001 的准确性：靠提取而非 Agent 自报，才可能真正记住"用户纠正过什么"。
- **DEC-008**（跨工具接力 E2E 为 CI 门禁）— REQ-001 的**直接验收测试**，挂了这个需求就没被满足。
