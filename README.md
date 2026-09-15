**English** | [汉语](./README.zh-CN.md)

# AlfredContext

**Git-versioned project memory for AI coding agents** — compiles low-token context so Codex, Claude Code and others pick up where the last one left off.

`alfredc` · `.alfredc/` · [Protocol v0.1](./spec/protocol.md)

---

## The problem

Quota limits force you to switch between AI coding agents. Codex today, Claude Code tomorrow, OpenCode after that.

Every new agent starts blind. It doesn't know what the previous one did. So you either burn tokens re-teaching it the whole project, or watch it repeat a mistake you already corrected two days ago.

What gets lost is the **detail**. Not "the login page is done" — but *"the user rejected modals for auth because of mobile UX, and that decision is still binding."*

Named after the butler: different Batmen, one Alfred.

## What AlfredContext is

Not another AI memory tool. A **project continuity and audit layer** that sits underneath whatever coding agent you happen to be using.

It records the development history as a Git-versioned ledger, and compiles from it the small, high-signal context a new agent actually needs.

Three words carry the weight here: **Continuity**, **Provenance**, **Context Compilation**.

## How it works

Event Sourcing + Materialized Context.

```
        Ledger — append-only, the source of truth
        .alfredc/history/2026-09-15.jsonl
                      │
                      │  compiled · regenerable at any time
                      ▼
        Materialized views — what the LLM reads
        brief.md · constraints.md · current.md
```

Every event carries an **authority level**, and can **supersede** earlier ones:

| Event type | Authority |
| --- | --- |
| `user_correction` | highest |
| `user_requirement` | ▲ |
| `accepted_decision` | │ |
| `project_fact` | │ |
| `verification` | │ |
| `agent_hypothesis` | lowest |

**This is what fixes the original problem.** When you reject an approach, a `user_correction` supersedes it. Every future agent sees the binding constraint *and the reason it exists* — not a summary that quietly dropped it.

Because the ledger is the truth and the views are derived, nothing is lost to repeated summarization. A new agent gets ~2,000 tokens of exactly what matters, instead of re-reading 500 files and 80k tokens of chat history.

## Zero model configuration

You do **not** configure a model for AlfredContext. Extraction reuses the coding agent you are already running — no API key, no extra account, no extra bill.

## Status

**Protocol-first. There is no code yet.**

| | |
| --- | --- |
| ✅ | **Protocol v0.1 finalized** — [`spec/protocol.md`](./spec/protocol.md) |
| ✅ | **Dogfooded** — this repository's own development history is kept in [`.alfredc/`](./.alfredc/) |
| ⬜ | **`alfredc` CLI not implemented** — the Go skeleton has not started |

The protocol is the deliverable right now. If you want to shape the format before it freezes, this is the moment.

## Repository layout

| Path | What it is |
| --- | --- |
| [`spec/protocol.md`](./spec/protocol.md) | **The public spec.** Normative — implementations follow this. |
| [`design/`](./design/) | Internal development docs. **Not** normative; may contain unverified research. |
| [`.alfredc/`](./.alfredc/) | This project's own memory — AlfredContext dogfooding itself. |
| [`friction-log.md`](./friction-log.md) | Every place the protocol felt awkward in real use. Doubles as the requirement backlog. |

## Supported agents

Planned for v1:

| Agent | Repo instructions | Status |
| --- | --- | --- |
| Claude Code | `CLAUDE.md` → `AGENTS.md` | ⬜ planned |
| Codex | `AGENTS.md` | ⬜ planned |
| OpenCode | `AGENTS.md` | ⬜ planned |
| ZCode | `AGENTS.md` | ⬜ planned |
| Qoder | `AGENTS.md` / `.qoder/rules` | ⬜ planned |

> ⚠️ **These integrations are planned, not verified.** The hook and headless-entry capabilities of each tool have not yet been checked against their official documentation. Tracked as `Q-001` in [`.alfredc/current.md`](./.alfredc/current.md).

## Non-goals

- **Not a model.** It never calls an LLM API and never holds credentials.
- **Not an MCP server** (v1).
- **Not a vector database** or semantic search.
- **Not a transcript archive.**
- **Not a replacement** for any coding agent.

## Documentation

- [Protocol v0.1](./spec/protocol.md) — the normative spec
- [Current state](./.alfredc/current.md) — what's done, what's next, open questions
- [Friction log](./friction-log.md) — where the design hurt, and what changed because of it
- [Design notes](./design/) — background and early exploration

## Contributing

There is no code to contribute to yet — which makes this the best time to argue with the design.

The protocol is in [`spec/protocol.md`](./spec/protocol.md). If something in it is wrong, ambiguous, or missing, an issue is more valuable right now than a pull request.

## License

[Apache-2.0](./LICENSE) — Copyright © 2026 ShawnZhang31.

Chosen deliberately. The explicit patent grant and permissive terms make it straightforward for companies to adopt AlfredContext and to implement the protocol in their own adapters — which is the entire point of publishing a spec.
