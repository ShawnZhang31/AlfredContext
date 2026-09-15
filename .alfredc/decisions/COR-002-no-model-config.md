---
id: COR-002
type: user_correction
status: active
created: 2026-09-15T11:00:00Z
source: claude-code
actor: user
scope: [spec/protocol.md]
supersedes: [DEC-011]
superseded_by: null
---

# COR-002 Do not require users to configure a model for AlfredContext

## Correction

不想在一个项目使用了 AlfredContext 之后，还需要给 AlfredContext 配置一个模型；而是用户当前使用什么 Coding Agent，就让用户正在使用的那个 Coding Agent 去提取。

## Reason

（用户未显式说明原因，从需求推断）AlfredContext 的核心卖点是**轻量 + 跨工具**。若使用前必须先申请/配置一个模型或 API key，就构成劝退式的前置成本——这与 REQ-001（让工具真正被用起来）和 DEC-002（轻量路线）直接冲突。

## Rejected Approach

**[DEC-011] 提炼引擎 = Claude Haiku 级模型 + 规则护栏**（已废止）

该方案技术上可行，但引入了"为工具配置模型"这一前置条件。

## Replacement

**[DEC-012] 提炼引擎 = 用户正在使用的 Coding Agent**（headless 调起）

`alfredc` 自身不持有任何凭据；提取由 Adapter 通过各工具 headless 模式完成。引擎降级阶梯：发起 session 的 Agent → 机器上任何已装 Agent → manifest 显式配置命令 → none（留在队列手工录入）。

### ⚠️ 由此产生的一条硬前提

本方案下，**提取者很可能就是干活的那个模型**。因此协议 §7.3 的三条铁律（归属可证、不确定即降级、永不静默改写）**必须由确定性代码执行，绝不能交给模型自觉**——否则本方案退化为已被否决的机制 A（Agent 自报）。

## Related

- REQ-001 / DEC-002（轻量）/ DEC-011（被废止）/ DEC-012（替代方案）
