---
id: DEC-007
type: accepted_decision
status: active
created: 2026-09-15T10:00:00Z
source: claude-code
actor: user
scope: [cmd/**, internal/**]
supersedes: []
superseded_by: null
---

# DEC-007 Implementation language is Go

## Decision

AlfredContext 的 CLI（`alfredc`）用 **Go** 实现。

## Reason

决策由**运行形态**决定，不是语言偏好：

| 约束 | Go 的满足方式 |
| --- | --- |
| Hook 高频调用，冷启动是硬指标 | 静态二进制冷启动 ~2-5ms（Node / Python 约 50ms 起） |
| 单二进制分发 | `CGO_ENABLED=0` + 纯 Go SQLite → 零依赖静态二进制 |
| 跨平台 | `goreleaser` 交叉编译 darwin / linux / windows × amd64 / arm64 |
| Adapter 语言无关 | 五工具 Hook 多为 shell 命令；唯一例外 OpenCode Plugin（TS）写 thin shim |

## Alternatives considered

- **Rust** — 冷启动更快，但开发效率低，对贡献者门槛高。
- **TypeScript / Bun** — 若需深度复用 OpenCode 的 TS 插件生态则更优；但 Bun `--compile` 产物较大、原生模块偶有坑，且冷启动不如 Go。
- **Python** — 分发困难（需解释器），Hook 冷启动最差，不适合此形态。

## 三条由此推导的硬约束

1. **禁止引入 cgo 依赖** —— 一旦启用 cgo，交叉编译与静态分发全部失效。SQLite 必须用 `modernc.org/sqlite`（纯 Go）。
2. **Hook 脚本必须是 thin wrapper** —— shell 一行调用 `alfredc`，不承载任何逻辑（含仓库根解析）。
3. **二进制体积与冷启动纳入 CI 门禁** —— 冷启动回归会直接伤害 Hook 体验。

## Related

- REQ-001（连续性依赖 Hook 可靠触发）/ DEC-008（测试策略）/ 协议 §12.1 / §12.2
