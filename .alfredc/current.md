<!-- generated: do not edit; use `alfredc record` -->
# Current

## Objective

定稿 AlfredContext v0.1 核心协议，并完成本仓库的**手工自举**——即用 AlfredContext 自己的格式管理 AlfredContext 的开发历史，以此验证协议是否顺手。

## Status

- ✅ 协议 v0.1 已定稿（`spec/protocol.md`），全部待定问题已关闭
- ✅ 文档分层已确立：`design/`（内部，不对外）+ `spec/`（公开规格，对外发布）
- ✅ 五个关键决策已定：机制 B、Push 注入、Go、五层测试、提炼引擎用现有 Coding Agent
- 🔄 手工自举进行中：账本（27 条事件）、派生视图、引导器
- ⬜ Go CLI 骨架未开始——尚无任何代码

## Next Action

1. 完成手工自举的收尾（引导器 + 摩擦日志）
2. 起 Go CLI 骨架，第一个命令 `alfredc init`
3. 实现 Context Compiler 与协议 §12.3 的第 2、4 层测试

## Open Questions

- **[Q-001]** 五工具 hook 对照表与 headless 入口（`claude -p` / `codex exec` / `opencode run`）**尚未逐条核实官方文档**，含二手来源。实现每个 Adapter 前必须核实，尤其 Codex 与 OpenCode 的注入能力。
- **公开规格的语言**：`spec/protocol.md` 目前是中文，但它是对外发布的**规范性**文档——第三方 Adapter 作者（可能不读中文）需要能读它。是否提供英文版，或改为英文为主 + 中文版另置？与 README 的双语方案（REQ-005）保持一致更合理。**发布前必须决定。**
- **文件改名后 `scope` 失效**（F-008 的长期方案）：账本一旦已提交，历史事件的路径失效需要编译器支持 rename 映射，或把 `scope` 改为语义化命名而非物理路径。v1 不实现。
- **ID 顺序号在团队协作下的冲突**：`key` 解决了幂等，但 `id` 顺序号在两个分支并行编写时仍会撞号。v1 不处理。
- **`CLAUDE.md` 符号链接的跨平台问题**：本仓库用 `CLAUDE.md -> AGENTS.md` 符号链接，Windows 下 git 检出会退化为普通文本文件而静默失效。`alfredc init` 应生成含 `@AGENTS.md` 的真实文件。
