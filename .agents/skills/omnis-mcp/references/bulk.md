# Bulk Omnis MCP work — delegating to subagents

**Use when delegating Omnis MCP work to subagents** (bulk class/method builds, slicing, monitoring, relaunches), and after any MCP version change (checklist at the end).

## Delegation rules

- **Context is the binding constraint** (each preview echoes the full diff). Slice by class type; each worker runs the preview→apply-next-turn pipeline, verifies its slice via `get_details` (line counts + structure) and reports per-method line counts; the parent cross-slice spot-checks and owns the goal.
- **Don't ping busy workers** — a ping can land mid-generation and interrupt an in-flight `preview_set_method` (whole batch re-staged); heavy-body generation takes minutes per turn. Rely on completion notices; probe progress read-only. Pings only at a step boundary with a genuine concern.
- **`subagent_fork` inherits the entire parent conversation** — a fat parent context (big spills/read-backs) ate a recorded worker's runway ("ran out of room", 0 methods applied). Keep the parent slim; prefer a fresh `subagent` for relaunches.
- **`"was stopped before it finished"` is a run-settled notice, not a death signal** — a misread spawned a duplicate worker on a still-running slice. Relaunch only when `list_agents` shows `[ready]` **and** progress 0; `[running]` = working, even if slow.
- **Progress probes give false stall alarms** (`search_library` index lag) — a recorded session raised two, both workers healthy on self-report. Trust `list_agents` status + worker self-reports; marker probes are never the sole basis for relaunching.
- **Never trust a memory-derived class list** (a recorded one had 5 wrong names) — fetch ground truth via class-level `get_details`.

## Re-verification checklist (after any MCP version change)

Re-probe before trusting the core skill:

1. Schema enums vs description text (`variableScope`/`type`/`action`).
2. Accepted values: `instance`/`local` scopes, non-`character` types, `itemref` workaround.
3. Token `expires` lifetime in practice.
4. Coercions (`short`/`0dp`/`datetime`) and the constants exception.
5. Tool surface: unified vs split read tools; test-execution tools present.
6. `lookup_symbols` / `omnis://docs` availability.
