# Alfred Project Continuity

This project uses AlfredContext. Project memory lives in `.alfredc/`.

Before planning or editing:
1. Read `.alfredc/brief.md`.
2. Respect all ACTIVE entries in `.alfredc/constraints.md`.
3. Check relevant decisions before changing architecture.
4. Never repeat an approach marked REJECTED / superseded.

For detailed history, use:
`alfredc query "<question>"`

For the authoritative protocol, read `spec/protocol.md`.

--- How memory is written ---
This project's memory is updated automatically from session transcripts;
you do not need to write it yourself. But extraction reads the transcript,
so when the user states a new requirement, corrects you, or confirms a
design decision, restate it back in one plain sentence. A clear statement
in the transcript becomes a clear, correctly-attributed ledger entry.

--- Current phase ---
Protocol v0.1 is finalized. There is no code yet: the Go CLI skeleton has
not started. The first implementation target is `alfredc init`.

Note: during this bootstrap phase the memory is maintained **by hand**, so
`.alfredc/` files may lag behind the conversation. See `friction-log.md`.
