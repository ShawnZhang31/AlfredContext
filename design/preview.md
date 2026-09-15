user:
由于很多AICoding工具如Codex、Claude Code、ZCode的plan又额度限制，使得我不得不在一个项目的开发过程中，不停的不同的AICoding工具或模型之前切换；这就使得切换AICoding工具或模型后，先的AICoding工具或模型，总是不知道之前的AICoding工具或模型做过啥，要么需要浪费大量的token重新熟悉整个项目，即便如此之前AICoding或模型与用户对话的细节、设计详细要求也会了解的不够，使得以前模型犯过的错误本来已经在用户的要求夏修正了，后面新的模型还有可能犯。
因此我想开发一个AICoding管家的工具，可以确保在项目不同的AICoing或同一个AICoing切换不同模型的时候，依然可以了解这个项目之前开发的细节问题。同时也可以让用户后续复核、审计AICoding设计和开发的细节。
我暂时也没有想好这个工具的形态，但是我喜欢这个工具应该是轻量级的，这个工具输出给AICoding或大模型看的内容，可以提交到代码仓库中，以便在团队协作中可以使用。我暂时将这个工具命名为Alfred，就是喜欢它可以像Alfred协助不同版本的蝙蝠侠一下，协助不同的AICoding工具更好的完成项目开发。
我有一个要求，要至少支持主流的AICoding工具，如Codex、Claude Code、OpenCode、ZCode、Qoder等。
现在你先帮我调研一下有没有类似的工具，帮我思考一下如何实现这工具。

chatgpt:
我调研了一圈后，结论很明确：**你的需求已经被很多人感知到了，但目前还没有一个主流工具把它完整解决好。** 市面上的方案大多只解决“Spec”“Memory”“Handoff”“Task”中的一部分。

我反而认为，你这个工具最有价值的定位不是“又一个 AI Coding Memory”，而是：

> **面向多 AICoding 工具的项目连续性与开发审计层。**
> 不替代 Codex / Claude Code / OpenCode / ZCode / Qoder，而是让它们像不同代的 Batman 一样，共享同一个 Alfred。

这个定位是成立的。

## 一、我找到的类似工具

目前最接近你想法的几类产品如下。

| 工具                       | 核心思路                                                                          | 和 Alfred 的相似度 | 主要缺口                            |
| ------------------------ | ----------------------------------------------------------------------------- | ------------: | ------------------------------- |
| **ShareMemory**          | 项目内 `AI_MEMORY/`，Claude Code 与 Codex 共享 decisions/tasks/learnings/corrections |         ★★★★★ | 当前主要聚焦 Claude+Codex，自动采集和审计能力有限 |
| **Agent Handoff**        | `.handoff/` 保存当前状态、决策、修改、验证结果，跨 Codex/Claude 接力                               |         ★★★★★ | 更像“交接班”，不是长期项目记忆体系              |
| **ai-memory**            | 自动采集 prompt/tool/decision，生成共享 Markdown Wiki，可跨 20+ Agent                     |         ★★★★★ | 架构更重，有 Server/MCP 色彩            |
| **coding-agent-toolkit** | `.workspace/` 保存 spec、plan、memory、transition，让 Claude/Codex 接着干               |         ★★★★☆ | 强工作流导向，不是通用审计/记忆层               |
| **OpenSpec**             | repo 中保存 proposal/spec/design/task，让多个 Agent 共享需求                             |         ★★★★☆ | 保存“应该做什么”，不保存“过去发生了什么”          |
| **GitHub Spec Kit**      | Spec → Plan → Tasks → Implement，支持大量 Coding Agent                             |         ★★★☆☆ | 更偏 Harness/SDD，流程较重             |
| **Serena**               | `.serena/memories/` 保存项目知识，MCP 提供代码语义能力                                       |         ★★★☆☆ | 更偏代码理解，不强调用户纠正和开发历史             |
| **Beads**                | 用依赖图式 Issue 代替易腐烂 Markdown Plan                                               |         ★★★☆☆ | 解决长任务状态，不解决完整上下文                |

其中 **ShareMemory** 和你的想法非常接近。它明确针对“Claude Code 与 Codex 互相不知道对方干过什么”的问题，在仓库中维护 `AI_MEMORY/`，包含 handoff、decision、task、learning、correction。([GitHub][1])

**Agent Handoff** 也很值得借鉴。它保存 current summary、next action、open tasks、key decisions、blockers、files touched、verification state，核心就是跨模型切换时不要重新读整个仓库。([GitHub][2])

更值得关注的是 **ai-memory**：它已经尝试自动捕获 prompt、tool call 和 decision，并将长期信息写成 Markdown Wiki；目前声称支持 Claude Code、Codex、OpenCode 等多种 Harness。([GitHub][3])

而 OpenSpec 和 GitHub Spec Kit 则证明了另一个关键点：**以 Markdown + Git 作为 AI Coding 工具之间的最低公共协议，是现实可行的。** OpenSpec 本身就通过 AGENTS.md 兼容 Codex、OpenCode 等工具；Spec Kit 目前已经提供几十种 Coding Agent 集成。([GitHub][4])

---

# 二、但我认为这些工具都没有完全解决你的核心问题

你描述的问题，其实不是简单的“Memory”。

假设你说：

> 登录页不要使用 Modal。

Agent A 一开始用了 Modal。

你纠正：

> 不行，因为移动端交互有问题，以后这个项目登录都不要 Modal。

Agent A 改掉了。

然后两天后切到 Agent B。

真正应该传递给 B 的不是：

> “登录页面已经完成。”

而应该是：

> **REQ-023：认证相关交互禁止 Modal。**
> 来源：用户明确要求。
> 原因：移动端交互问题。
> 曾经尝试 Modal → 用户否决。
> 当前状态：Active。
> 相关代码：`src/auth/*`。
> 优先级：高于 Agent 自己提出的 UI 方案。

这已经不是聊天摘要。

这是一个：

**Project Decision / Requirement / Correction Ledger。**

我认为这是 Alfred 最应该抓住的差异。

---

# 三、我建议 Alfred 不要做成“大模型记忆系统”

我更建议采用：

> **Event Sourcing + Materialized Context**

也就是：

**完整历史是一套事件账本，给模型看的 Context 是根据账本动态生成的“物化视图”。**

这是整个 Alfred 最关键的架构思想。

例如某次开发产生事件：

```text
USR-018  User Requirement
用户要求日志模块统一使用 slog

AGT-031  Agent Decision
Agent 决定封装 logger abstraction

USR-021  User Correction
用户否决 logger abstraction
原因：项目规模较小，不希望过度设计

DEC-014  Accepted Decision
直接使用 slog，不增加 wrapper

VER-009  Verification
go test ./... passed
```

以后换成任何模型，都不需要把整个聊天记录重新塞进去。

Alfred 只需要告诉它：

```text
Active constraint:
- Use slog directly.
- Do NOT introduce logger abstraction.
- This was explicitly rejected by the user.
See: USR-021 / DEC-014.
```

这就非常省 Token。

而如果用户将来问：

> 当初为什么没有做 Logger 抽象？

Alfred 又能把完整因果链找出来。

这就是你说的**复核、审计**。

---

# 四、我建议仓库里的 Alfred 长这样

核心仍然全部是普通文件：

```text
my-project/
├── AGENTS.md
├── CLAUDE.md
│
├── .alfred/
│   ├── brief.md
│   ├── requirements.md
│   ├── constraints.md
│   ├── current.md
│   │
│   ├── decisions/
│   │   ├── DEC-001-auth.md
│   │   └── DEC-002-storage.md
│   │
│   ├── learnings/
│   │   ├── frontend.md
│   │   └── testing.md
│   │
│   ├── tasks/
│   │   └── TASK-023.md
│   │
│   └── history/
│       ├── 2026-09-05.jsonl
│       └── 2026-09-06.jsonl
│
└── .git/
```

这里要刻意区分两种东西。

**`.alfred/history/*.jsonl` 是事实账本，不应该随便重写。**

而：

**`brief.md / current.md / constraints.md` 是为 LLM 优化的 Materialized View，可以不断重新生成。**

这样就解决了一个非常麻烦的问题：

> 摘要压缩多次以后，细节不断丢失。

原始事件还在，所以随时可以重建。

---

# 五、Alfred 最重要的模块应该叫 Context Compiler

我甚至认为它比 Memory 更重要。

假设：

```bash
alfred context
```

输出：

```text
# Alfred Project Brief

Current objective
Implement OAuth login.

Current status
Backend callback completed.
Frontend login flow in progress.

Critical user constraints
1. Do not use modal-based authentication.
2. Do not add another state-management library.
3. All API errors must use existing AppError.

Relevant decisions
DEC-012 OAuth provider abstraction accepted.
DEC-018 JWT storage uses HttpOnly cookie.

Rejected approaches
COR-007 localStorage token storage
Reason: explicitly rejected by user.

Known pitfalls
LEARN-004 Vitest requires TZ=UTC.

Next action
Implement src/auth/LoginPage.tsx.

Recommended files to read
- src/auth/*
- DEC-012
- DEC-018
```

控制在比如：

**1000～2500 Tokens。**

而不是：

> 新模型进来先读 500 个文件 + 8 万 token 聊天历史。

这是 Alfred 真正能给你省钱的地方。

进一步可以：

```bash
alfred context --task "实现 OAuth 登录"
```

只返回和 OAuth、auth、相关代码路径有关的记忆。

我第一版甚至**不会上 Vector DB**。

直接：

Git + Markdown + YAML Frontmatter + SQLite 本地索引 + ripgrep

完全够用。

`.alfred/cache.db` 放 `.gitignore`。

仓库里的 Markdown 才是真相来源。

---

# 六、最关键的是跨 AICoding 工具怎么接

这里情况比我原来预想的还要乐观。

你点名的几个产品都已经具备不错的接入点。

| 工具          | Repo 指令                      | 自动 Hook / Plugin | Alfred 适配方式            |
| ----------- | ---------------------------- | ---------------- | ---------------------- |
| Codex       | `AGENTS.md`                  | Hooks            | AGENTS + Hook          |
| Claude Code | `CLAUDE.md`                  | Hooks            | CLAUDE → Alfred + Hook |
| OpenCode    | `AGENTS.md`                  | Plugin Events    | AGENTS + Plugin        |
| ZCode       | `AGENTS.md`                  | Hooks            | AGENTS + Hook          |
| Qoder       | `AGENTS.md` / `.qoder/rules` | Hooks            | AGENTS + Hook          |

Qoder 已经原生兼容 `AGENTS.md`，而且项目 Rules 本身就可以提交 Git。([Qoder Docs][5])

ZCode 也直接支持 Workspace `AGENTS.md`，同时还有自己的 Project Memory。([ZCode][6])

OpenCode 当前更是明确把 `AGENTS.md` 当作项目级持久指令，并推荐提交 Git。([OpenCode][7])

Claude Code 是唯一稍微不同的，它原生使用 `CLAUDE.md`，但官方文档明确建议，如果项目已经有 `AGENTS.md`：

```text
CLAUDE.md

@AGENTS.md
```

就可以共享同一套规则。([Claude][8])

所以我建议：

> **AGENTS.md 做 Alfred 的 Universal Bootloader。**

而不是把所有 Memory 写进 AGENTS.md。

例如：

```markdown
# Alfred Project Continuity

This project uses Alfred.

Before planning or editing:
1. Read `.alfred/brief.md`.
2. Respect all ACTIVE entries in `.alfred/constraints.md`.
3. Check relevant decisions before changing architecture.
4. Never repeat an approach marked REJECTED.
5. Before finishing work, update Alfred project state.

For detailed history, use:
`alfred query "<question>"`
```

这几十行就够了。

---

# 七、自动采集其实也已经具备条件

这是我觉得这个项目现在非常值得做的原因。

Claude Code 的 Hook 可以拿到 `transcript_path`，Stop 阶段还能拿到 `last_assistant_message`；也就是说 Alfred 可以在每轮结束以后自动分析这一轮到底发生了什么。([Claude][9])

ZCode 的 Hook 同样提供 transcript path，而且为了插件兼容，同时提供 Claude Code 风格的 snake_case 字段。([ZCode][10])

Qoder 的 Hook 生命周期包括 SessionStart、UserPromptSubmit、PostToolUse、Stop、SessionEnd、PreCompact 等，而且还公开了 `QODER_TRANSCRIPT_PATH`。([Qoder Docs][11])

OpenCode 则提供 message、session、file、tool 等大量 Plugin Event。([OpenCode][12])

所以可以形成：

```text
               ┌─────────────┐
Claude Code ──▶│             │
Codex ────────▶│             │
OpenCode ─────▶│ Alfred      │
ZCode ────────▶│ Adapters    │
Qoder ────────▶│             │
               └──────┬──────┘
                      │
                  Normalized
                    Events
                      │
                      ▼
              ┌───────────────┐
              │ Alfred Ledger │
              └───────┬───────┘
                      │
              Context Compiler
                      │
           ┌──────────┴──────────┐
           ▼                     ▼
     brief/context          audit/history
           │
           ▼
      Next AI Coding
```

我会把 **Adapter 与 Core 完全分开**。

以后 Cursor、Gemini CLI、Kiro、Copilot CLI 出现变化，只加 Adapter。

---

# 八、一定要给“记忆”设计权威等级

这是很多 Memory 产品容易做错的地方。

Agent 自己说：

> 我认为这里应该使用 Repository Pattern。

绝对不能因此变成：

> 项目要求使用 Repository Pattern。

所以 Alfred 内部至少要区分：

```text
USER_REQUIREMENT
USER_CORRECTION
ACCEPTED_DECISION
PROJECT_FACT
AGENT_HYPOTHESIS
FAILED_ATTEMPT
VERIFICATION
OPEN_QUESTION
HANDOFF
```

而优先级可以大致是：

```text
后续用户明确纠正
    >
用户明确要求
    >
用户确认的设计决策
    >
已合并项目规范
    >
代码事实
    >
Agent 推断
```

尤其应该有：

```yaml
id: COR-018
type: user_correction
status: active
supersedes:
  - DEC-012
scope:
  - src/auth/**
reason: Mobile UX
```

**supersedes 是一个非常重要的设计。**

它可以真正解决你说的：

> “旧模型已经犯过一次错误，用户纠正了，新模型又重新犯一次。”

---

# 九、我建议第一版不要做 MCP Server

MCP 很诱人，但我不建议它成为 Alfred 的基础。

因为你的目标是：

> 轻量 + 跨工具 + Git 可协作。

所以优先级应该是：

**CLI > 文件协议 > Hooks > MCP。**

MCP 应该是后面的增强能力：

```text
alfred.search_memory
alfred.get_decision
alfred.get_history
alfred.record_decision
alfred.explain
```

没有 MCP 的 Coding Agent，一样能：

```bash
alfred context
alfred query "为什么认证没有用 localStorage"
```

这才真正跨平台。

---

# 十、我会这样做 MVP

第一阶段只做 6 个命令就够了：

1. `alfred init`：生成 `.alfred/`、AGENTS.md 和各工具 Adapter。
2. `alfred context`：生成 token-budgeted 项目上下文。
3. `alfred record`：从当前 Session 提炼 requirement / correction / decision / learning / verification。
4. `alfred query "..."`：查询历史原因、要求、踩坑记录。
5. `alfred handoff`：生成当前开发交接状态。
6. `alfred audit`：按时间线展示“用户要求 → Agent 决策 → 修改 → 验证”。

第一批就支持你要求的：

**Codex + Claude Code + OpenCode + ZCode + Qoder。**

而且这五个产品现在的 Hook / AGENTS 能力已经足以做一个比较漂亮的第一版。

---

# 十一、还有一个很现实的问题：Alfred 这个名字已经撞车了

我特意查了一下。

目前已经有一个比较活跃的开源项目就叫 **Alfred**，它做的还是 AI Coding：支持 Claude Code、Codex、OpenCode，把多个 Coding Engine 组织成一个自动化软件工程团队，同时也有 memory、audit、context 等能力。([GitHub][13])

此外还有另一个 `alfred-agent`，主打可验证自治、inspectable memory 和 replayable ledger。([GitHub][14])

所以如果这是你内部研发项目，**Alfred 完全可以继续作为代号**。

但如果未来准备开源，我会建议最终变成类似：

**Alfred Context / Alfred Ledger / Alfred Butler / Alfred Memory Protocol**

或者另起正式名称。

否则 GitHub / npm / SEO 都会非常痛苦。

---

# 我对这个产品最终形态的判断

我不建议把 Alfred 做成：

> “一个帮 AI 记东西的软件。”

而应该定义成：

> **Alfred 是独立于 Coding Agent 的项目连续性协议与审计运行时，将开发过程中产生的需求、纠正、决策、尝试、验证和工作状态沉淀为 Git 可版本化的项目记忆，并按任务动态生成低 Token 上下文，使 Codex、Claude Code、OpenCode、ZCode、Qoder 等不同 Coding Agent 能够无缝接力。**

这里面我认为真正有产品壁垒的是三个词：

**Continuity / Provenance / Context Compilation**

连续性、可追溯、上下文编译。

不是 Memory 本身。

如果按你的目标继续往下推进，**下一步最值得做的不是直接写代码，而是把 Alfred v0.1 的「Memory Schema + Event Schema + `.alfred/` 目录规范 + 五种 Coding Agent Adapter 协议」定下来。** 这实际上就是 Alfred 的核心协议；CLI 反而只是外壳。

[1]: https://github.com/ycl-2004/ShareMemory?utm_source=chatgpt.com "GitHub - ycl-2004/ShareMemory: Project-scoped shared memory for Claude Code and Codex AI coding agents. · GitHub"
[2]: https://github.com/AniruddhaHumane/handoff?utm_source=chatgpt.com "GitHub - AniruddhaHumane/handoff: Save and resume coding-agent context across Codex, Claude, and parallel agent workflows. · GitHub"
[3]: https://github.com/akitaonrails/ai-memory?utm_source=chatgpt.com "GitHub - akitaonrails/ai-memory: Solution for long term memory for agent coding CLIs and to facilitate handoff between different agent vendors · GitHub"
[4]: https://github.com/qkeluna/openspec?utm_source=chatgpt.com "GitHub - qkeluna/openspec: Spec-driven development for AI coding assistants. · GitHub"
[5]: https://docs.qoder.com/user-guide/rules?utm_source=chatgpt.com "Rules - Qoder"
[6]: https://zcode.z.ai/en/docs/agents?utm_source=chatgpt.com "ZCode Agent | ZCode Docs"
[7]: https://opencode.ai/v2/docs/instructions?utm_source=chatgpt.com "Instructions | OpenCode"
[8]: https://code.claude.com/docs/en/memory?utm_source=chatgpt.com "How Claude remembers your project - Claude Code Docs"
[9]: https://code.claude.com/docs/en/hooks "Hooks reference - Claude Code Docs"
[10]: https://zcode.z.ai/en/docs/hooks "Hooks | ZCode Docs"
[11]: https://docs.qoder.com/extensions/hooks?utm_source=chatgpt.com "Hooks - Qoder"
[12]: https://opencode.ai/docs/plugins/?utm_source=chatgpt.com "Plugins | OpenCode"
[13]: https://github.com/luminik-io/alfred?utm_source=chatgpt.com "GitHub - luminik-io/alfred: An autonomous engineering team that ships while you're away. Built on Claude Code and Codex. · GitHub"
[14]: https://github.com/BeamusWayne/Alfred?utm_source=chatgpt.com "GitHub - BeamusWayne/Alfred: A verifiable autonomous coding agent (CLI) — TypeScript on Bun. Machine-enforced done-gates, agent-curated inspectable memory, signed replayable run ledgers. 850+ tests. Try it: bunx alfred-agent demo · GitHub"
