---
title: AlfredContext Protocol
version: 0.1
schema_version: "0.1"     # 与 .alfredc/manifest.yaml 对应，标识磁盘格式兼容性
status: draft
updated: 2026-09-15
visibility: public        # 本文件是公开规格，随产品对外发布
---

# AlfredContext Protocol v0.1

> **本文档是公开规格。** 它定义 `.alfredc/` 的磁盘格式与 Adapter 接口契约——**任何人都可以据此实现自己的 Adapter**，无需阅读 AlfredContext 的实现代码。
> 状态：Draft（内容已定稿，尚未发布 1.0）
> 范围：目录规范、Event Schema、Memory Schema、权威等级、Context Compiler、五种 Coding Agent 的 Adapter 协议、CLI 命令规范。
> 原则：**协议先行，CLI 只是外壳。**

## 0. 文档权威

本项目有两类文档，性质不同，**不要混淆**：

| 位置 | 性质 | 是否对外 | 是否规范 |
| --- | --- | --- | --- |
| **`spec/`**（本文件） | 产品的**公开格式规格** | ✅ 对外发布 | ✅ **规范性**——实现以此为准 |
| `design/` | 内部研发文档（想法探讨、未验证调研、内部笔记） | ❌ 不对外 | ❌ 非规范性——仅作背景 |
| `.alfredc/` | 本项目**自己的**开发记忆（dogfooding） | ❌ 不对外 | ❌ 非规范性——是数据，不是规格 |

> 边界原则：**内部文档可以包含未验证内容，公开规格不行。**
> 因此 `design/preview.md` 中来自调研的第三方工具对比、命名撞车结论等**未经逐条核实**的内容，只存在于 `design/`，不进入 `spec/`。
> 实现与本文档冲突时，**以本文档为准**。

**许可**：本项目采用 [Apache-2.0](../LICENSE)。**你可以自由地依据本规格实现自己的 Adapter，包括用于商业目的**——发布一份规格的全部意义就在于让它被实现。

---

## 1. 命名与身份

| 项 | 值 |
| --- | --- |
| 项目名 | **AlfredContext** |
| 命令 | **`alfredc`**（如 `alfredc init`），避免与已有 `alfred` 包冲突 |
| 目录 | **`.alfredc/`** |
| 使命 | 独立于 Coding Agent 的项目连续性协议与审计运行时 |

---

## 2. 定位与目标

### 2.1 一句话定位

> AlfredContext 将开发过程中产生的**需求、纠正、决策、尝试、验证、工作状态**沉淀为 **Git 可版本化的项目记忆**，并按任务**动态生成低 Token 上下文**，使 Codex、Claude Code、OpenCode、ZCode、Qoder 等不同 Coding Agent 无缝接力。

### 2.2 三个护城河词

- **Continuity（连续性）**：换工具/换模型，不重新熟悉项目。
- **Provenance（可追溯）**：任何一条约束都能追到「谁、何时、为什么」。
- **Context Compilation（上下文编译）**：不是塞入全部历史，而是按需编译出精炼上下文。

### 2.3 目标（Goals）

1. 轻量：无 Server、无 Vector DB，`Git + Markdown + JSONL + SQLite 本地索引`。
2. **零模型配置**：不要求用户为 AlfredContext 申请/配置任何模型或 API key，提取复用用户正在使用的 Coding Agent（§12.4）。
3. 跨工具：通过 `AGENTS.md` + 各工具 Hook/Plugin 接入，Adapter 与 Core 分离。
4. Git 可协作：所有真相来源都是可提交的普通文件。
5. 可审计：完整因果链可回溯。

### 2.4 非目标（Non-goals，v1 明确不做）

- ❌ 不做 MCP Server（作为后续增强）。
- ❌ **不自带、不托管、不代理任何模型**——AlfredContext 永不是一个"AI 产品"，它只调度用户已有的 Coding Agent（§12.4）。
- ❌ 不做 Vector DB / 语义检索（`ripgrep` + 关键词 + 结构化索引足够）。
- ❌ 不做聊天记录全文存档（原始 transcript 默认 gitignore，仅保留提炼后的事件）。
- ❌ 不替代任何 Coding Agent 的调度/执行能力。

---

## 3. 核心架构原则

### 3.1 Event Sourcing + Materialized Context

```
                    事实账本（append-only，真相来源）
   ┌──────────────────────────────────────────────┐
   │  .alfredc/history/2026-09-14.jsonl            │
   │  {id, type, source, actor, ...}  ← 每行一个事件 │
   └──────────────────────┬───────────────────────┘
                          │  动态编译（可反复重建）
                          ▼
   ┌──────────────────────────────────────────────┐
   │  物化视图（Materialized View，给 LLM 看）       │
   │  brief.md / constraints.md / current.md       │
   │  decisions/*.md / learnings/*.md              │
   └──────────────────────────────────────────────┘
```

### 3.2 铁律（最重要的规则）

> **Ledger 是真相，View 是派生。**

- `.alfredc/history/*.jsonl` 是**事实账本**：只追加，不重写；是唯一真相来源。
- `constraints.md / current.md / decisions/*.md / learnings/*.md` 是**派生视图**：由 `alfredc` 从账本编译生成，可随时重建。
- **不要手工编辑派生视图**——改了也会在下一次编译时被覆盖。要改，就用 `alfredc record` 写入一条新事件，再由编译器重建视图。
- 唯一例外：`brief.md` 是「半手工」文件——由 `alfredc init` 初始化，用户/Agent 可编辑，但它表达的是项目长期愿景，不随单次会话变化。
- **不可变性的边界 = 账本的第一次提交。** 在账本首次提交到版本控制之前，启动阶段的修正（路径、拼写、字段错误）允许**就地改正**；**一旦提交，账本即成为历史，只允许追加**，修正错误必须靠追加带 `supersedes` 的新事件。

> 为什么边界是"第一次提交"而非"写入之时"：提交前无人依赖这份账本，改它不伤害任何人；提交后它就是审计依据，改它就破坏审计。**发布即是不可变性的分界。**
> （此规则由手工自举发现——文件重组导致历史事件的 `scope` 路径失效，见 `friction-log.md` F-008 与 `DEC-015`。）

这条铁律直接解决「多轮摘要压缩后细节不断丢失」的问题：原始事件还在，随时可重建。

---

## 4. `.alfredc/` 目录规范

```text
my-project/
├── AGENTS.md               # Universal Bootloader（见 §10）
├── CLAUDE.md               # 仅 Claude Code：@AGENTS.md 桥接
│
├── .alfredc/
│   ├── manifest.yaml       # 协议版本 + 项目元信息
│   ├── brief.md            # 项目简报（稳定，半手工）
│   ├── constraints.md      # Active 约束（派生，按权威降序）
│   ├── current.md          # 当前状态 + 下一步（派生）
│   ├── requirements.md     # 需求清单视图（派生）
│   │
│   ├── decisions/
│   │   └── DEC-<id>-<slug>.md
│   ├── learnings/
│   │   └── <topic>.md      # 按主题聚合的踩坑记录
│   ├── tasks/
│   │   └── TASK-<id>.md
│   │
│   ├── history/
│   │   ├── 2026-09-14.jsonl    # 事件账本（提交到 Git）
│   │   └── transcripts/        # 归一化 transcript（gitignore）
│   │
│   ├── queue/              # 待提取队列（§9.1 阶段一产物，gitignore）
│   │   └── <session>.json
│   │
│   └── cache.db            # 本地 SQLite 索引（gitignore）
└── .git/
```

### 4.1 提交 vs 忽略

| 路径 | 是否提交 | 说明 |
| --- | --- | --- |
| `AGENTS.md` / `CLAUDE.md` | ✅ 提交 | 引导器 |
| `.alfredc/manifest.yaml` | ✅ 提交 | 协议元信息 |
| `.alfredc/brief.md` | ✅ 提交 | 项目简报 |
| `.alfredc/constraints.md` 等派生视图 | ✅ 提交 | 让新工具/新队友无需跑 CLI 也能直接读 |
| `.alfredc/decisions/` `learnings/` `tasks/` | ✅ 提交 | 同上 |
| `.alfredc/history/*.jsonl` | ✅ 提交 | 审计账本，真相来源 |
| `.alfredc/history/transcripts/` | ❌ 忽略 | 归一化对话，可能含敏感信息；提取产物的溯源锚 |
| `.alfredc/queue/` | ❌ 忽略 | 待提取队列，本地中间态，可重建 |
| `.alfredc/cache.db` | ❌ 忽略 | 本地索引，可重建 |

### 4.2 `manifest.yaml`

```yaml
schema_version: "0.1"
project: "my-project"
created_at: "2026-09-14T10:00:00Z"
enabled_adapters: [claude-code, codex, opencode, zcode, qoder]
id_counters:            # 人类可读 ID 的顺序计数器（机器幂等靠 key，见 §9.7）
  REQ: 0
  COR: 0
  DEC: 0
  # ...
```

---

## 5. Event Schema（账本）

`.alfredc/history/YYYY-MM-DD.jsonl`，每行一个 JSON 对象（NDJSON）。

### 5.1 通用字段

```jsonc
{
  "id": "COR-018",                  // 人类可读 ID（§5.2）
  "key": "a1c4f0e2b9...",           // 机器幂等键 = sha1(session|turn|type|summary)，见 §9.7
  "type": "user_correction",        // 事件类型（见 §5.3）
  "ts": "2026-09-14T10:00:00Z",     // RFC3339 UTC
  "source": "claude-code",          // 产生此事件的工具
  "session": "sess_8f2a...",        // Coding Agent 会话 ID（溯源 + 幂等用）
  "turn": 7,                        // 源自第几轮（幂等键组成）
  "actor": "user",                  // "user" | "agent" —— 决定权威级，见 §7.3
  "origin": "extracted",            // extracted | manual —— 自动提炼 or 人工录入
  "confidence": "high",             // high | medium | low —— 提炼置信度，见 §7.3
  "scope": ["src/auth/**"],         // 影响的路径/域，glob（供 --task 过滤与注入匹配）；`["**"]` = 项目全局
  "summary": "Auth flows must not use modals",   // 英文一句话，进上下文（省 token，见 §12.4）
  "detail": "移动端交互问题，用户明确要求登录相关不使用 Modal",  // 中文详述，按需展开
  "priority": "critical",           // critical | high | normal | low
  "status": "active",               // active | superseded | obsolete | open
  "supersedes": ["DEC-012"],        // 本事件废止的 ID
  "superseded_by": null,            // 本事件被谁废止（编译时回填）
  "evidence": {                     // 溯源，二选一，取决于 origin
    "transcript": "history/transcripts/sess_8f2a.jsonl",   // origin: extracted
    "lines": [42, 58]                                      //   指向归一化 transcript 的行（§9.3）
    // "note": "用户在会话中明确要求"                       // origin: manual —— 来源说明（§7.3 铁律 1）
  },
  "extraction": {                   // 提炼元数据（幂等用，§9.7）
    "transcript_hash": "sha256:...",
    "extractor": "claude-code",      // 执行提取的引擎（Coding Agent 名，见 §12.4）
    "extracted_at": "2026-09-14T10:05:00Z"
  }
}
```

> **`scope` 约定**：`["**"]` 表示**项目全局**。空数组等价于 `["**"]`，`alfredc` 统一规范化为 `["**"]`。
> （由手工自举发现——全局决策此前没有表示法，见 `friction-log.md` F-003）

### 5.2 ID 前缀 ↔ 事件类型

| 前缀 | type | 含义 |
| --- | --- | --- |
| `REQ` | `user_requirement` | 用户明确要求 |
| `COR` | `user_correction` | 用户纠正（含对 Agent 决策/先前方案的否决） |
| `DEC` | `accepted_decision` | 双方确认的决策 |
| `FACT` | `project_fact` | 项目客观事实 |
| `HYP` | `agent_hypothesis` | Agent 未确认的推断 |
| `FAIL` | `failed_attempt` | 试过但失败/被否的方法 |
| `VER` | `verification` | 验证结果（测试/构建通过等） |
| `Q` | `open_question` | 待决问题 |
| `HANDOFF` | `handoff` | 会话交接状态 |

### 5.3 事件类型枚举

| type | 权威级 | 典型触发 | 举例 |
| --- | --- | --- | --- |
| `user_correction` | 最高 | 用户否决/纠正 Agent 或先前决策 | 「不要 Modal」「别加 logger 抽象」 |
| `user_requirement` | 高 | 用户提出新需求 | 「日志统一用 slog」 |
| `accepted_decision` | 中高 | 双方敲定方案 | 「OAuth provider 用抽象层」 |
| `project_fact` | 中 | 客观事实 | 「本仓库 monorepo，Go 1.22」 |
| `verification` | 中 | 测试/构建结果 | 「`go test ./...` passed」 |
| `agent_hypothesis` | 低 | Agent 提议未确认 | 「这里也许该用 Repository 模式」 |
| `failed_attempt` | 记录型 | 失败或被否的方案 | 「localStorage 存 token → 被否」 |
| `open_question` | 记录型 | 悬而未决 | 「是否需要国际化？」 |
| `handoff` | 记录型 | 会话结束 | 当前状态 + 下一步 |

> 说明：`failed_attempt` 与 `user_correction` 常成对出现——前者记录「试过 X 失败」，后者记录「用户因此定下规则」。二者都要留痕：前者回答「为什么没这么做」，后者回答「以后都不许这么做」。

> ⚠️ **谁能写哪种事件，不由本表决定**——`actor` 字段（user / agent）才决定权威级，且提炼器必须过 §7.3 的**撰写权限闸门**。Agent 说「应该用 X 模式」绝不等于项目要求。

### 5.4 示例（一段真实开发产生的事件流）

```jsonl
{"id":"REQ-018","key":"a1c4f0e2","type":"user_requirement","ts":"2026-09-14T09:00:00Z","source":"claude-code","session":"sess_a1","turn":1,"actor":"user","origin":"extracted","confidence":"high","scope":["src/log/**"],"summary":"Use slog for all logging","detail":"日志模块统一使用 slog","priority":"high","status":"active","supersedes":[],"superseded_by":null}
{"id":"HYP-031","key":"7d2b91af","type":"agent_hypothesis","ts":"2026-09-14T09:12:00Z","source":"claude-code","session":"sess_a1","turn":4,"actor":"agent","origin":"extracted","confidence":"high","scope":["src/log/**"],"summary":"Wrap logger behind an abstraction for future swap","detail":"","priority":"low","status":"superseded","supersedes":[],"superseded_by":"COR-021"}
{"id":"COR-021","key":"e5f81c30","type":"user_correction","ts":"2026-09-14T09:20:00Z","source":"claude-code","session":"sess_a1","turn":6,"actor":"user","origin":"extracted","confidence":"high","scope":["src/log/**"],"summary":"Rejected the logger abstraction","detail":"项目规模较小，不希望过度设计","priority":"critical","status":"active","supersedes":["HYP-031"],"superseded_by":null}
{"id":"DEC-014","key":"c93ae077","type":"accepted_decision","ts":"2026-09-14T09:25:00Z","source":"claude-code","session":"sess_a1","turn":7,"actor":"user","origin":"extracted","confidence":"high","scope":["src/log/**"],"summary":"Call slog directly; no wrapper","detail":"由 COR-021 推导，用户当轮明确确认","priority":"high","status":"active","supersedes":[],"superseded_by":null}
{"id":"VER-009","key":"2f60d5b4","type":"verification","ts":"2026-09-14T09:40:00Z","source":"claude-code","session":"sess_a1","turn":9,"actor":"agent","origin":"extracted","confidence":"high","scope":["src/log/**"],"summary":"go test ./... passed","detail":"","priority":"normal","status":"active","supersedes":[],"superseded_by":null}
```

> 注意 `HYP-031` 与 `DEC-014` 的区别：前者是 Agent 自己提的方案（`actor: agent`，不进 `constraints.md`），后者**必须**有用户当轮的明确确认语句才允许落为 `accepted_decision`，否则按 §7.3 铁律 2 降级为 `agent_hypothesis`。

---

## 6. Memory Schema（物化视图）

所有视图文件用 **Markdown + YAML Frontmatter**。派生视图文件头带 `<!-- generated -->` 标记。

### 6.1 `brief.md`（半手工，稳定）

```markdown
---
id: brief
updated: 2026-09-14T10:00:00Z
---
# Project Brief
## What
## Why
## Tech Stack
## Non-goals
```

### 6.2 `constraints.md`（派生，最关键的视图）

编译所有 `status: active` 的事件，**按权威降序**排列，每条回溯到 ID。

```markdown
<!-- generated: do not edit; use `alfredc record` -->
# Active Constraints

## Critical — User Corrections
1. [COR-021] Do not introduce a logger abstraction — use slog directly
   - source: user_correction · reason: project is small; avoid over-engineering
   - detail: 项目规模较小，不希望过度设计
   - supersedes: HYP-031

## High — User Requirements
2. [REQ-018] Use slog for all logging in this module

## Accepted Decisions
3. [DEC-014] Call slog directly; no wrapper

## Project Facts
4. [FACT-003] This repo is a Go module; `go test ./...` is the entry point

---

## Superseded（仅供审计，不参与注入）
- ~~[HYP-031] Wrap logger behind an abstraction~~ → superseded by **COR-021**
```

> **分组顺序即权威降序**（§7.1）。示例必须覆盖全部五个分组——**示例即规范，读者会照着示例做而不是照着规则表做**。
> （由手工自举发现：§6.2 原示例只演示了 Critical / High，导致 `project_fact` 被漏编，见 `friction-log.md` F-007）

> 视图按 §12.5 的语言约定编译：**硬约束用英文**（进上下文、省 token），**`detail` 保留中文**（供人复核时看原文意思）。

> **Superseded 段是必需的**（由手工自举发现，见 `friction-log.md` F-002）。
> 若只列 Active，读者只看到「不许用 X」，看不到「曾用过 X，因 Y 被否」——而后者才是审计价值所在，也正是 §7.1 `supersedes` 链存在的意义。
> **边界**：Superseded 段**不参与 Context 注入**（§9.4：注入只含 Active 且 `actor: user` 的事件），它纯粹供人查阅。

### 6.3 `current.md`（派生）

```markdown
<!-- generated -->
# Current
## Objective
## Status
## Next Action
## Open Questions
```

### 6.4 `decisions/DEC-<id>-<slug>.md`

每个 `accepted_decision` / 重要的 `user_correction` 生成一个文件。

```markdown
---
id: DEC-014
type: accepted_decision
status: active
created: 2026-09-14T09:25:00Z
source: claude-code
scope: [src/log/**]
supersedes: []
superseded_by: null
---
# DEC-014 Call slog directly; no wrapper

## Decision
## Reason
## Alternatives considered
## Related
- REQ-018 / COR-021
```

### 6.5 `learnings/<topic>.md`（按主题聚合踩坑）

```markdown
---
topic: testing
updated: 2026-09-14T10:00:00Z
---
# Learnings: testing

## LEARN-004 Vitest requires TZ=UTC
- 现象 / 原因 / 解决
- 关联事件：FAIL-004 / COR-007
```

### 6.6 `requirements.md`（派生，需求清单）

按状态分组列出所有 `user_requirement`，便于需求审计。

---

## 7. 权威等级与 supersedes 语义

### 7.1 权威排序（高 → 低）

```text
后续用户明确纠正 (user_correction, 越新越高)
    >
用户明确要求 (user_requirement)
    >
用户确认的设计决策 (accepted_decision)
    >
已合并项目规范 (project_fact)
    >
验证结果 (verification)
    >
Agent 推断 (agent_hypothesis)
```

### 7.2 supersedes 规则

- 一条事件可用 `supersedes: [id, ...]` 废止其它事件；被废止事件 `status → superseded`，其 `superseded_by` 回填。
- **权威约束**：废止方权威必须 ≥ 被废止方（`user_correction` 可废止 `accepted_decision`；`agent_hypothesis` 不可废止 `user_requirement`）。
- 生效规则：一个命题若存在 `supersedes` 链，只有链顶（未被废止者）进入 `constraints.md`。

> 这正是解决你最初痛点的机制：「旧模型犯过的错被用户纠正了，新模型又犯一次」——因为 `constraints.md` 里那条 `user_correction` 是 Active，且带 `supersedes` 链，新模型会被强制看到「X 已被否决，原因是 Y」。

### 7.3 撰写权限与归属矩阵

> 前提：v0.1 采用**机制 B（事后提炼）**——账本由 Core 的提取器从 transcript 提炼写入，而非 Agent 自报。
> 于是必须回答一个 §5 没回答的问题：**提炼器可以自动落账哪些事件？**

**第一原则：权威由归属决定，不由措辞决定。**

Agent 说过「应该用 Repository 模式」，不构成项目要求；只有 `actor: user` 的语句才有高权威。**提炼器最容易犯的错，就是把 Agent 的提议误判成用户的要求或确认。**

| 事件类型 | 可否自动落账 | 进 `constraints.md` | 判定依据 |
| --- | :---: | :---: | --- |
| `user_requirement` | ✅ | ✅ | transcript 中**用户本人**的需求陈述 |
| `user_correction` | ✅ | ✅ | transcript 中**用户本人**的否决/纠正 |
| `accepted_decision` | ⚠️ 条件 | ✅（仅 `confidence: high`） | 需定位到用户的**明确确认语句**；否则降级 |
| `project_fact` | ✅ | ✅ | 代码/配置等客观事实 |
| `verification` | ✅ | ❌（进 audit） | 命令输出 |
| `agent_hypothesis` | ✅ | ❌ | Agent 原话 |
| `failed_attempt` | ✅ | ❌（进 Rejected 段） | 失败/被否的方案 |
| `open_question` | ✅ | ❌ | 悬而未决 |
| `handoff` | ✅ | ❌ | 会话交接 |

**三条铁律：**

1. **归属必须可证**（**仅适用于 `origin: extracted`**）：任何由提炼产生的 `actor: user` 事件，其 `evidence.lines` 必须指向 transcript 中用户的原始语句。**定位不到 → 强制降级为 `agent_hypothesis`。** 这是防止「凭空制造最高权威约束」的闸门——没有它，`supersedes` 机制反而会放大错误（一条伪造的 `user_correction` 能顺带废止正确决策）。
   > **人工录入（`origin: manual`）不受此约束**：作者身份由构造保证（是用户自己敲的），无需引用。以 `evidence.note` 记录来源说明即可。
   > 此限定由手工自举发现——自举时没有 transcript 可引用，按原字面执行会把全部手工账本强制降级。见 `friction-log.md` F-001 与 `DEC-013`。
2. **不确定即降级**：`accepted_decision` 若无法定位用户的明确确认语句，降级为 `agent_hypothesis` 并置 `confidence: low`。**宁可漏记，不可错记**——一条错误的 Active 约束会主动误导后续所有 Agent，比缺失更糟。
3. **永不静默改写**：提炼器只能**追加**事件。纠正错误必须追加一条带 `supersedes` 的新事件（与 §3.2 铁律一致），**不得直接修改已有事件**（账本 append-only）。

> 抽查机制：`alfredc review` 列出 `confidence != high` 或 `actor: user` 的事件，供用户确认/否决（见 §11）。这是对「自动落账」的必要安全网。

---

## 8. Context Compiler 规范

最重要的模块。`alfredc context` 从账本 + 视图编译出低 Token 上下文。

### 8.1 输出结构（固定顺序）

```text
# Alfred Project Brief
Current objective
Current status
Critical user constraints      ← 权威降序，仅 Active
Relevant decisions             ← 与当前任务相关
Rejected approaches            ← failed_attempt / superseded
Known pitfalls                 ← learnings
Next action
Recommended files to read
```

### 8.2 Token 预算

- 默认 **2000 tokens**，可 `--budget N` 调整（范围建议 1000~2500）。
- 编译时按「权威 → 优先级 → 时间」截断，超预算部分降级为「见 `alfredc query`」。

### 8.3 任务过滤

```bash
alfredc context --task "实现 OAuth 登录"
```

- 用关键词 + `scope` 路径匹配（`ripgrep`），只返回与 OAuth/auth 相关的记忆。
- 无 `--task` 时返回全量 Active 约束 + 最近状态。

### 8.4 示例输出（对齐 preview.md）

```text
# Alfred Project Brief

Current objective
Implement OAuth login.

Current status
Backend callback completed. Frontend login flow in progress.

Critical user constraints
1. Do not use modal-based authentication.      (COR-007)
2. Do not add another state-management library. (REQ-012)
3. All API errors must use existing AppError.   (COR-015)

Relevant decisions
DEC-012 OAuth provider abstraction accepted.
DEC-018 JWT storage uses HttpOnly cookie.

Rejected approaches
COR-007 localStorage token storage — explicitly rejected by user.

Known pitfalls
LEARN-004 Vitest requires TZ=UTC.

Next action
Implement src/auth/LoginPage.tsx.

Recommended files to read
- src/auth/*
- DEC-012, DEC-018
```

---

## 9. Adapter 协议

### 9.1 两阶段提取流水线（机制 B）

机制 B 下提炼需要一次模型调用（秒级），而 **Hook 绝不能阻塞 Agent**。因此流水线必须分两阶段：

```
[Agent 会话进行中]
      │  工具原生 Hook 触发（Stop / SessionEnd）
      ▼
阶段一 · Enqueue（同步，必须 < 50ms，绝不调用模型）
  Adapter 读取工具原生 transcript
    → 归一化为 Normalized Transcript（§9.3）
    → 落盘 .alfredc/history/transcripts/<session>.jsonl
    → 写一条待处理记录到 .alfredc/queue/<session>.json
      ▼
阶段二 · Extract（异步，任何一次 alfredc 调用时顺带触发）
  Core 检查队列 → 调起**用户正在使用的 Coding Agent**（§12.4）跑提取
    → 候选 Event → 过 §7.3 权限闸门 → 追加进账本
    → 重建派生视图（§6）
```

- **不阻塞**：Agent 在 Stop 时立即返回，提取稍后完成。
- **可重入**：提取中断可重跑，不影响账本（幂等，见 §9.7）。
- **可手动**：`alfredc record` 立即处理队列，并支持交互式补充。
- **零模型配置**：提取引擎就是用户已在用的 Coding Agent，**AlfredContext 自身不持有任何模型凭据**（§12.4）。
- **可离线**：提取不在会话热路径上，可在任意时刻、用任意可用引擎完成。

### 9.2 Core 与 Adapter 的职责划分

机制 B 带来的最大架构收益：**提取逻辑不再分散到五个 Adapter，而是 Core 共享一份。**

| 职责 | 归属 | 说明 |
| --- | --- | --- |
| transcript 格式归一化 | **Adapter**（每工具一份） | 纯格式翻译，**不做语义判断** |
| 语义提取（这轮发生了什么） | **Core**（唯一一份） | §9.1 阶段二 |
| 权限闸门与落账 | Core | §7.3 |
| 派生视图编译 | Core | §6 / §8 |
| 上下文注入 | **Adapter**（每工具一份） | 各工具注入点不同，见 §9.4 |

> 结论：**Adapter 变薄、Core 变厚。** 新增一个工具 = 写一个格式翻译器 + 一个注入器，**不涉及任何语义逻辑**。这也保证五个工具用同一把尺子判断「什么算 user_requirement」。

### 9.3 Normalized Transcript Schema（Adapter ↔ Core 的接口契约）

这是 Adapter 与 Core 之间**唯一的接口**，也是新增 Adapter 时唯一需要对齐的东西。

```jsonc
{
  "session": "sess_8f2a",
  "source": "claude-code",            // 归一化前的工具
  "started_at": "2026-09-14T09:00:00Z",
  "ended_at":   "2026-09-14T09:40:00Z",
  "cwd": "/Users/x/proj",
  "turns": [
    {"i": 1, "role": "user",  "text": "登录页不要用 Modal"},
    {"i": 2, "role": "agent", "text": "好的，我改用独立页面"},
    {"i": 3, "role": "tool",  "name": "Edit",
     "target": "src/auth/Login.tsx", "ok": true,
     "digest": "将 Modal 替换为独立路由页"}      // 摘要，非全参数
  ],
  "raw_ref": "history/transcripts/sess_8f2a.jsonl"  // 原始文件指针
}
```

三条设计约束：

1. **只保留语义相关内容**：工具调用的完整参数（文件全文、命令输出）可能极大。`digest` 只留摘要 + `target` 路径，**体积可降一个数量级**，直接决定提取成本。
2. **`raw_ref` 是溯源的锚**：Event 的 `evidence.lines` 指向该原始文件的行号，供审计回溯到原话。
3. **Adapter 不做语义判断**：`turns` 是忠实翻译。「这条算不算 `user_requirement`」是 Core 的事——保证五个工具**同一把尺子**。

### 9.4 注入协议（Push 为主，Pull 为辅）

**Push 注入点**（按 Hook 生命周期）：

| 触发点 | 注入内容 | 预算 | 目的 |
| --- | --- | --- | --- |
| **SessionStart** | 完整 context（§8） | ≤ 2000 tok | 新会话开局即知全局 |
| **UserPromptSubmit** | **仅当 prompt 命中某 scope 的 Active 约束时**，注入该 scope 的约束摘要 | ≤ 300 tok | 精准拦截「又要犯的错」 |
| **PreCompact** | 完整 context 重新注入 | ≤ 2000 tok | **对抗摘要压缩导致的细节丢失** |
| 会话中按需 | Agent 自行 `alfredc query` | — | Pull 兜底 |

⚠️ **UserPromptSubmit 不能每轮全量注入**——那会每轮烧掉 2000 tokens，并把 Agent 淹没在重复信息里。**只在命中 scope 时注入短约束**，是成本与效果的最优点。

> **PreCompact 这一条值得单独强调。** Agent 上下文被压缩的那一刻，正是 Alfred 重新注入「不可丢失约束」的最佳时机——压缩会把细节磨掉，而约束必须活下来。这直接命中你最初的痛点。Claude Code 与 Qoder 均提供该 hook（能力核实见 §9.5）。

> ⚠️ **Push 的安全边界**：注入内容来自账本，而账本可能含 Agent 写的文本（`agent_hypothesis`）。**注入默认只包含 Active 且 `actor: user` 的约束**，绝不把 Agent 的猜测当规则回灌给下一个 Agent。

### 9.5 五工具接入点

| 工具 | Repo 指令 | 采集（Enqueue） | 注入（Push） | transcript 来源 |
| --- | --- | --- | --- | --- |
| **Claude Code** | `CLAUDE.md` → `@AGENTS.md` | Hook `Stop` / `SessionEnd` | Hook `SessionStart` / `UserPromptSubmit` / `PreCompact` | `transcript_path` |
| **Codex** | `AGENTS.md` | Hooks（❓待核实） | Hooks（❓待核实） | 待核实 |
| **OpenCode** | `AGENTS.md` | Plugin `session` / `message` 事件 | Plugin 事件 | Plugin API |
| **ZCode** | `AGENTS.md` | Hooks（兼容 Claude 风格 snake_case 字段） | Hooks | transcript path |
| **Qoder** | `AGENTS.md` / `.qoder/rules` | `Stop` / `SessionEnd` | `SessionStart` / `UserPromptSubmit` / `PreCompact` | `QODER_TRANSCRIPT_PATH` |

> ⚠️ **核实说明**：本表的 hook 名称与能力来自调研资料（含二手来源），**尚未逐一比对官方文档**。实现每个 Adapter 前必须逐个核实，尤其 Codex 与 OpenCode 的注入能力。
> 协议只约定 Adapter 的**职责**（归一化 + 注入），**不绑定具体 hook 名**——这是协议能跨工具稳定的前提。

### 9.6 Adapter 契约

每个 Adapter 必须实现：

1. **`normalize(raw_transcript) -> NormalizedTranscript`** —— 格式翻译，见 §9.3。
2. **`enqueue(session)`** —— 阶段一，**必须 < 50ms 且绝不调用模型**。
3. **`inject(mode)`** —— 按 §9.4 在三个注入点写入内容；`mode ∈ {session_start, prompt_submit, pre_compact}`。
4. **仓库根解析** —— Hook 的工作目录**不保证是仓库根**。实现前必须 `git rev-parse --show-toplevel` 解析，否则 `.alfredc/` 相对路径会错位。

### 9.7 幂等与去重

机制 B 下「同一个 transcript 被提取两次」是常态（重跑、并行触发、崩溃恢复），必须幂等：

- **幂等键**：以 `session` 为单位。同一 session 重复提取必须产出**完全相同的事件集**，不重复落账。
- **实现**：事件携带 `key`（`sha1(session | turn | type | summary)`）与 `extraction.transcript_hash`。已提取且 hash 未变 → 跳过；hash 变了（会话被追加内容）→ 仅提取新增部分。
- **约束**：事件 ID 必须**稳定可复现**，否则幂等失效。

> 这条把「ID 方案」从风格选择升级为**幂等性前提**，并直接给出结论——**双身份**：
> - `id`：人类可读的顺序号（`COR-021`），用于引用与讨论；
> - `key`：内容派生的稳定键，用于机器幂等与并发合并。
>
> 二者并存，各司其职。（原待定问题「ID 方案」就此关闭，见 §13 决策记录。）

---

## 10. AGENTS.md Universal Bootloader

`alfredc init` 生成。几十行，是所有工具的公共入口：

```markdown
# Alfred Project Continuity

This project uses AlfredContext. Project memory lives in `.alfredc/`.

Before planning or editing:
1. Read `.alfredc/brief.md`.
2. Respect all ACTIVE entries in `.alfredc/constraints.md`.
3. Check relevant decisions before changing architecture.
4. Never repeat an approach marked REJECTED / superseded.

For detailed history, use:
`alfredc query "<question>"`

--- How memory is written ---
This project's memory is updated automatically from session transcripts;
you do not need to write it yourself. But extraction reads the transcript,
so when the user states a new requirement, corrects you, or confirms a
design decision, restate it back in one plain sentence. A clear statement
in the transcript becomes a clear, correctly-attributed ledger entry.
```

Claude Code 的 `CLAUDE.md` 仅含 `@AGENTS.md` 一行，实现共享同一套规则。

> **机制 B 的副作用**：既然账本由 transcript 提炼而来，**Agent 在对话中把用户的要求/纠正复述清楚，就能显著提升提炼准确率**。最后那段 "How memory is written" 就是为此设计的——它把「提炼质量」变成了 Agent 可以主动配合的事，而不是纯靠事后猜测。

---

## 11. CLI 命令规范

**MVP 六个命令：**

| 命令 | 作用 | 输入 | 输出 |
| --- | --- | --- | --- |
| `alfredc init` | 生成 `.alfredc/`、`AGENTS.md`、`CLAUDE.md`、各工具 Adapter | — | 目录骨架 + 引导器 |
| `alfredc context [--task] [--budget]` | 编译低 Token 上下文 | 可选任务描述 | §8 输出 |
| `alfredc record` | 处理提取队列，写入事件（可选交互补充） | 队列 / 交互式 | 新 Event → 重建视图 |
| `alfredc query "<q>"` | 查历史原因/要求/踩坑 | 自然语言 | 因果链 + 关联事件 |
| `alfredc handoff` | 生成交接状态 | — | current.md + HANDOFF 事件 |
| `alfredc audit` | 时间线展示「要求→决策→修改→验证」 | 可选时间/范围 | 审计视图 |

**建议追加第 7 个（机制 B 的安全网）：**

| 命令 | 作用 | 为什么需要 |
| --- | --- | --- |
| `alfredc review` | 列出 `confidence != high` 或 `actor: user` 的事件，供确认/否决 | 自动落账必须有抽查出口，否则错误的 Active 约束无人发现（§7.3） |

> `alfredc context` / `record` / `review` 在启动时都会顺带**检查并处理提取队列**——这让「异步提取」在用户无感知的情况下自然发生。

---

## 12. 技术栈与实现选型

### 12.1 语言与运行时：Go

选型由**运行形态**决定，不是语言偏好——`alfredc` 会被 Hook **高频调用**，冷启动是硬指标。

| 约束 | Go 的满足方式 |
| --- | --- |
| Hook 高频调用，冷启动是硬指标 | 静态二进制冷启动 **~2-5ms**（Node / Python 约 50ms 起） |
| 单二进制分发 | `CGO_ENABLED=0` + 纯 Go SQLite → **零依赖静态二进制**，`brew` / `npm` wrapper / `curl \| sh` 均可分发 |
| 跨平台 | `goreleaser` 交叉编译 darwin / linux / windows × amd64 / arm64 |
| Adapter 语言无关 | 五工具 Hook 多为 shell 命令；唯一例外是 OpenCode Plugin（TS），写一个 thin shim 转发给 `alfredc` |

**由此推导的三条硬约束：**

1. **禁止引入 cgo 依赖** —— 一旦启用 cgo，交叉编译与静态分发全部失效。SQLite 必须用纯 Go 实现。
2. **Hook 脚本必须是 thin wrapper** —— shell 一行调用 `alfredc`，**不承载任何逻辑**（含路径解析，见 §9.6）。
3. **二进制体积与冷启动纳入 CI 门禁** —— 冷启动回归会直接伤害 Hook 体验，必须被测试挡住。

### 12.2 依赖选型（v1 最小集）

| 用途 | 选型 | 备注 |
| --- | --- | --- |
| SQLite | `modernc.org/sqlite` | 纯 Go，免 cgo（**硬约束**，见 §12.1） |
| CLI 解析 | 标准库 `flag` + 子命令分发 | 命令才 7 个，避免 cobra 这类重依赖 |
| JSONL | 标准库 `encoding/json` | 逐行 NDJSON |
| glob 匹配 | `github.com/bmatcuk/doublestar` | 支持 `scope` 的 `src/auth/**` 语义 |
| 全文检索 | 外部 `ripgrep`（可选） | 缺失时降级为内置字符串匹配，**不设为硬依赖** |
| 测试 | 标准库 `testing` + 见 §12.3 | |

> 依赖克制是刻意的：**这是一个要活很多年的协议实现，依赖越少，被生态变化拖累的概率越低。**

### 12.3 测试策略

这个项目没有传统意义的"业务逻辑"，**它的质量 = 协议正确性 + 提炼准确性**。因此测试分五层，其中第 2、4 层是本项目独有：

| # | 层 | 测什么 | 手段 | 优先级 |
| --- | --- | --- | --- | :---: |
| 1 | **账本不变量** | `supersedes` 无环、不可越权废止、`superseded_by` 回填完整 | **属性测试**（`pgregory.net/rapid`）：随机生成事件图，断言不变量恒成立 | ⭐⭐⭐ |
| 2 | **Context Compiler 黄金文件** | 同一账本 → 同一输出；token 不超预算 | 黄金文件测试（`-update` 模式） | ⭐⭐⭐⭐⭐ |
| 3 | **Adapter 契约** | 五工具真实 transcript → 期望 Event | **sanitized transcript 语料库**，每工具一份 fixture | ⭐⭐⭐⭐ |
| 4 | **跨工具接力 E2E** | **产品核心承诺**：A 会话落账 → B 取 context → B 必须看到该约束 | 端到端：临时 git 仓库 → init → record → context → 断言 | ⭐⭐⭐⭐⭐ |
| 5 | **Adapter 一致性套件** | 第三方新 Adapter 是否合规 | 做成**公开 conformance kit** | ⭐⭐⭐⭐（战略级） |

**两个关键判断：**

1. **第 4 层是验收测试。** 你最初的痛点——"新模型不知道旧模型干过啥、又犯同样的错"——直接对应第 4 层的一个用例。**它挂了，产品就是失败的**，其余全白测。建议设为 CI 必过门禁。
2. **第 3 层的语料库是资产，不只是测试。** 五工具的真实（脱敏）transcript 是别人抄不走的护城河——它同时是**测试数据**、**`extract` 的评测集**、以及**新 Adapter 的参考实现**。

**工具**：标准库 `testing` + `github.com/rogpeppe/go-internal/testscript`（Go 官方自用的 CLI 测试框架，天然适合 `alfredc` 这种"命令 + 文件系统"形态）。

### 12.4 提炼引擎：用你正在使用的 Coding Agent

**核心原则：AlfredContext 不拥有模型，也不配置模型。**

`alfredc` 自身**不调用任何 LLM API、不持有任何凭据**。提取由「用户当前正在使用的那个 Coding Agent」在其 **headless 模式**下完成。

```
阶段二 · Extract
  alfredc 读 queue 里的 Normalized Transcript（§9.3）
    → 用「提取提示词 + transcript」调起 Coding Agent 的 headless 模式
        Claude Code → claude -p
        Codex       → codex exec
        OpenCode    → opencode run
        ZCode/Qoder → 各自的 headless 入口（❓待核实，同 §9.5）
    → 收回结构化输出（Event 候选）
    → 过 §7.3 确定性闸门 → 追加进账本
```

`alfredc init` **不询问任何 API key 或模型名**。这是"轻量"的核心，也避免了"为了用 AlfredContext，先得再去申请一个模型账号"的劝退设计。

**引擎选择的降级阶梯：**

| 优先级 | 引擎 | 说明 |
| :---: | --- | --- |
| 1 | **发起该 session 的 Adapter 自己的 Agent** | 最贴合「用户正在使用」 |
| 2 | 机器上任何已安装且支持 headless 的 Agent | 前者不可用时兜底 |
| 3 | `manifest.yaml` 显式配置的命令 | 逃生舱，见下 |
| 4 | `none` —— 事件留在队列 | 由用户 `alfredc record` 手工录入 |

> 这个阶梯能成立，**正是因为 §9.3 的 Normalized Transcript 是 agent-agnostic 的**——任何 Agent 都能读同一份输入。接口契约在这里兑现了价值。这也意味着：**用户换了 Coding Agent，队列里的历史 session 仍然可提取。**

**逃生舱（可选，非必需）：**

```yaml
extractor:
  mode: auto          # auto（默认）| agent:<name> | command | none
  # agent: claude-code                                       # 锁定某个 Agent
  # command: "my-llm --prompt {prompt} --input {transcript}"  # 自带廉价模型
```

#### ⚠️ 为什么这没有退化成「Agent 自报」

必须回答的质疑：提取交给 Coding Agent 做，不就退回机制 A（Agent 自报）了吗？

**不是。四点区别：**

| 维度 | 机制 A（已否决） | 本方案 |
| --- | --- | --- |
| 会话 | **同一个会话**，Agent 评价自己 | **全新独立会话** |
| 输入 | Agent 的记忆与意图 | **仅**归一化 transcript |
| 动机 | 倾向把自己的决定报成"已确认" | 不参与执行，只做判定 |
| 守门 | 无，靠 Agent 自觉 | **§7.3 闸门由确定性代码执行** |

**最后一行是关键**：本方案下，提取者很可能就是干活的那个模型。因此 **§7.3 的三条铁律必须由代码实现，绝不交给模型自觉**——「`actor: user` 必须有原话引用」这类规则不能随模型状态漂移。

> 这条从「好设计」升级为**本方案成立的前提**。若把闸门交给模型自觉，本方案就真的退化成机制 A 了。

#### 诚实的代价

| 代价 | 量级 / 缓解 |
| --- | --- |
| 消耗该 Agent 的额度 | **规模小且可界定**：只读一个 session 的归一化 transcript（§9.3 已压掉一个数量级），不是整个项目。换来的收益是后续所有会话省下大量 token——**投入产出比为正** |
| 要求 CLI 支持 headless | 需逐个核实（与 §9.5 同样的核实义务）；核实不了就落到阶梯第 3、4 级 |
| 比直连 API 慢 | 提取不在会话热路径上，慢无妨 |
| Agent CLI 接口/版本会变 | 正是 §12.3 第 3 层 Adapter 契约测试要挡的东西 |

### 12.5 数据约定

- **`cache.db` 边界：纯索引。** 只存可重建的加速结构（事件倒排、scope 索引），**不存任何派生视图内容**——避免出现第二份事实来源（§3.2 铁律）。
- **字段语言：`summary` 英文 + `detail` 中文。** `summary` 进上下文，英文省 token 且模型召回更好；`detail` 保留中文原文，供你复核时看准原意。派生视图（§6）同样遵循：**硬约束用英文，`detail` 保留中文**。

> 语言约定的例外：`detail` 中若含用户原话，**一律保留原始语言不做翻译**——翻译会损失"用户到底怎么说的"这一审计信息。

---

## 13. 决策记录（Decision Log）

> 按时间倒序。每条决策都应能在本协议中找到对应章节；实现时若与本节冲突，**以正文为准**。

| 日期 | 决策 | 章节 |
| --- | --- | --- |
| 2026-09-15 | **技术栈 = Go**（纯 Go SQLite、免 cgo、静态单二进制） | §12.1 / §12.2 |
| 2026-09-15 | **测试 = 五层策略**，跨工具接力 E2E 设为 CI 必过门禁 | §12.3 |
| 2026-09-15 | **提炼引擎 = 用户正在使用的 Coding Agent**（headless 调起，零模型配置）；§7.3 闸门由代码实现是本方案成立的前提 | §12.4 |
| 2026-09-15 | **`cache.db` = 纯索引**（不存派生视图，避免第二事实来源） | §12.5 |
| 2026-09-15 | **字段语言 = `summary` 英文 + `detail` 中文** | §12.5 |
| 2026-09-15 | **ID 方案 = 双身份**（人类可读 `id` + 内容派生 `key`），由机制 B 幂等要求推导 | §9.7 |
| 2026-09-15 | **注入方式 = Push 为主，Pull 为辅**（SessionStart / UserPromptSubmit / PreCompact） | §9.4 |
| 2026-09-15 | **账本写入机制 = B（事后提炼）**：Hook 采集 transcript，Core 提取落账，Agent 不自报 | §9.1 / §7.3 |
| 2026-09-14 | **撰写权限闸门**：`actor: user` 的事件必须可指向用户原话，否则降级 | §7.3 |
| 2026-09-15 | **引用规则仅适用于 `origin: extracted`**；人工录入的作者身份由构造保证 | §7.3 |
| 2026-09-15 | **`constraints.md` 增设 Superseded 段**（仅供审计，不参与注入） | §6.2 |
| 2026-09-15 | **`scope` 约定**：`["**"]` 表示项目全局，空数组规范化为此值 | §5.1 |
| 2026-09-14 | **目录 = `.alfredc/`**（非 `.alfred/`），命令 `alfredc`，项目名 **AlfredContext** | §1 |
| 2026-09-14 | **README = `README.md`（英文）+ `README.zh-CN.md`**，顶部各放语言选择链接 | — |

---

## 14. 下一步

协议 v0.1 已定稿。接下来：

1. **手工自举（立即可做）** —— 在本仓库按本协议手工创建 `.alfredc/`，把上面这张决策表逐条录成真实事件，验证 schema 是否顺手。摩擦记入 `friction-log.md`。
2. **起 Go CLI 骨架** —— `alfredc init` 第一个命令。
3. **实现 Context Compiler + 第 2、4 层测试** —— 先把"产品核心承诺"跑通，再补 Adapter。

---

[1]: https://code.claude.com/docs/en/memory "Claude Code memory docs"
[2]: https://code.claude.com/docs/en/hooks "Claude Code hooks"
[3]: https://zcode.z.ai/en/docs/hooks "ZCode hooks"
[4]: https://docs.qoder.com/extensions/hooks "Qoder hooks"
[5]: https://opencode.ai/docs/plugins/ "OpenCode plugins"
[6]: https://opencode.ai/v2/docs/instructions "OpenCode instructions"
[7]: https://docs.qoder.com/user-guide/rules "Qoder rules"
