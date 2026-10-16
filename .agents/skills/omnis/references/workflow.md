# Omnis Work — Operational Workflow (Tool-Specific Supplement)

Supplement to [`SKILL.md`](SKILL.md). All Omnis *language* rules live in SKILL.md; this file only covers how to operate the tooling around them. It is written for the official Omnis MCP (preview/apply staged-change tooling); the delegation section applies to any agent setup. If Omnis code is being edited through raw files (e.g. a `.omh` text export), only the delegation section applies.

## 1. Staged-change (preview/apply) tooling

- **Preview → apply within the same turn.** Staged previews live only ~2 minutes (shorter windows have been observed) and there is no refresh path. When a token expires, the apply fails with "No pending operation found" — recover by **re-previewing** the affected methods; re-previewing an unapplied *new* method is safe and idempotent.
- **Bulk pipeline**: apply the previous turn's batch and preview the next batch **in the same turn**; every token is consumed on its very next turn. The bridge handles large parallel batches (24 preview/apply calls per turn and 10 class-creations per turn have run clean). If one call in a parallel batch fails on transport while siblings succeed, retry just that one call verbatim — a failed staging leaves no half-state.
- **`variables` array discipline**: every entry needs an explicit `action` field. When re-staging a method whose parameters already exist, **omit the `variables` list entirely** — re-declaring existing parameters echoes them as "add" and risks duplicate declarations on apply.
- **Check the returned diff first.** If the method diff comes back empty, build a manual before/after from a method-details read rather than assuming nothing changed.
- **Treat tool schemas as a floor, not a ceiling.** Served enums under-report accepted values (scopes: `instance`/`local`/`parameter`; types: `boolean`/`character`/`date`/`number`/`binary`/`integer`/`object`/`itemref`; action lists). Probe empirically and cross-check against each tool's own description text.
- **Read paths**: a table class's object listing can come back empty — read its fields from the corresponding **schema class** instead. Object-level methods (e.g. a component's `$event`) must be addressed with the separate `method` and `object` parameters — the Studio-style `obj/$event` slash path is not accepted.
- **No method-reorder API**: new methods are appended at the end of the method list; placement is cosmetic.
- **Folders**: create them with `create_class` using `classType:"folder"`.
- **No execution / UI-driving**: run-verification in Studio (e.g. running a test suite) must be performed manually by the user — finish with read-backs and hand over a checklist (see SKILL.md, "Debugging and verification").
- **After any transport/session disruption**, re-verify the current library state with read-only calls before resuming writes — implement from verified ground truth, not from the prepared spec.

## 2. Bulk work, delegation, and context economics

- **Context is the budget.** Each method preview echoes the full diff back; hundreds of full-body rewrites exceed one session's context. Split the work into slices (fresh subagent per slice), have each worker verify its own slice (line counts + structure read-backs), and let the parent cross-check slices and own the goal.
- **Never poll busy subagents.** A status message arriving mid-generation can interrupt an in-flight staging call and force the worker to re-stage its whole batch. Rely on completion notices; probe progress read-only (e.g. search for a unique per-class marker string).
- **Library-search results lag** (index/timing): a fresh probe can return 0 matches for a just-applied class. Do not use search probes as the sole basis for declaring a stall or relaunching a worker.
- **Liveness of record**: trust the agent-status list (`running` = still working, even if slow) and worker self-reports. A "was stopped before it finished" notice is a **run-settled event for one turn**, not a death signal — never relaunch on that notice alone; treat a worker as dead only when its status is `ready` **and** progress is 0. Always confirm no worker is still active before relaunching, or you will create a duplicate on the same slice.
- **Context inheritance**: a forked subagent inherits the entire parent conversation, so a heavy parent context eats the worker's runway. For relaunches prefer a fresh (non-forked) subagent with a context-economy protocol: fewer previews per turn, marker-string searches instead of per-method full read-backs, and a progress file appended per batch. Keep the parent's own context slim so future forks start lighter.
- **Relaunches must be idempotent** (per-item marker probes), and the item list must come from the library (a ground-truth class listing), never from memory — memory-derived lists get corrupted.
