<!-- generated: do not edit; use `alfredc record` -->
<!-- 由 .alfredc/history/*.jsonl 编译。编译规则见协议 §6.2 / §7.1。 -->
# Active Constraints

> 按权威降序。仅收录 `status: active` 且 `actor: user` 的事件。
> 手工自举阶段由人工编译；`alfredc context` 实现后自动生成。

## Critical — User Corrections

1. **[COR-002] Do not require users to configure a model for AlfredContext**
   - Use whichever coding agent the user is already running to do the extraction
   - supersedes: DEC-011 · scope: `spec/protocol.md`
   - detail: 不想在一个项目使用了 AlfredContext 之后，还需要给 AlfredContext 配置一个模型；而是用户当前使用什么 Coding Agent，就让用户正在使用的那个 Coding Agent 去提取。

2. **[COR-001] Use `.alfredc/`, not `.alfred/`**
   - Avoid clashing with existing `alfred` packages; keep dir name consistent with the `alfredc` command
   - supersedes: HYP-001 · scope: `.alfredc/**`
   - detail: 项目的目录不要使用 .alfred/，而是使用 .alfredc/，避免和其他的 alfred 库冲突。

## High — User Requirements

3. **[REQ-001] Cross-agent project continuity and an audit trail**
   - New tools must not re-learn the project, and must not repeat errors the user already corrected
   - scope: `**` · detail: 切换工具后新工具不知道之前的工具做过什么，要么浪费大量 token 重新熟悉项目，要么重犯已被用户纠正过的错误。

4. **[REQ-002] Project name is AlfredContext; CLI command is `alfredc`**
   - scope: `**`

5. **[REQ-003] Must support at least Codex, Claude Code, OpenCode, ZCode, Qoder**
   - scope: `**`

6. **[REQ-004] Output must be committable to the repo for team collaboration**
   - scope: `**`

7. **[REQ-005] English README + `README.zh-CN.md` with a language selector**
   - scope: `README.md`, `README.zh-CN.md`

## Accepted Decisions

8. **[DEC-001] Protocol first; the CLI is only a shell** — scope: `spec/**`
9. **[DEC-002] Lightweight: no MCP server, no vector DB, no transcript archive** — scope: `**`
10. **[DEC-003] Ledger write mechanism B: post-hoc extraction from transcripts** — scope: `spec/protocol.md`
11. **[DEC-004] Injection is push-primary, pull-secondary** — 3 push points: SessionStart / UserPromptSubmit / PreCompact
12. **[DEC-005] Authorship gate: `actor: user` events must cite the user's original words**
13. **[DEC-006] Dual-identity IDs: human-readable `id` + content-derived `key`**
14. **[DEC-007] Implementation language is Go** — hard constraints: no cgo, thin-wrapper hooks, cold-start CI gate
15. **[DEC-008] Five-layer test strategy; cross-agent handoff E2E is a CI gate**
16. **[DEC-009] `cache.db` is a pure index** — never a second source of truth
17. **[DEC-010] Field language: English `summary`, Chinese `detail`** — user quotes never translated
18. **[DEC-012] Extraction engine is the user's active coding agent, invoked headless** — gate must stay in code
19. **[DEC-013] Citation rule applies only to `origin: extracted`, not manual entries**
20. **[DEC-014] Split repo docs: `design/` is internal, `spec/` is public** — the protocol is a *product spec*, not an internal doc; it must be publicly readable so third parties can write adapters
21. **[DEC-015] Immutability boundary of the ledger is its first commit** — bootstrap corrections allowed before; append-only after

## Project Facts

22. **[FACT-001] `spec/protocol.md` is the single source of truth for the protocol**
   - `design/preview.md` is the original brainstorm (background, not spec). On conflict, the protocol wins.
   - scope: `spec/**`

---

## Superseded（不再生效，保留供审计）

- ~~[HYP-001] Use `.alfred/` as the project memory directory~~ → superseded by **COR-001**
- ~~[DEC-011] Extraction engine is a Claude Haiku-class model~~ → superseded by **COR-002**
