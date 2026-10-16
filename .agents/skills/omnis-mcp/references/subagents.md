# Omnis MCP bulk work and delegation

Use this reference for Omnis MCP subagent slicing, monitoring, and relaunches. For raw-file edits such as `.omh` exports, only the delegation and context-management guidance applies. After an MCP version change, use the availability checklist at the end.

## Delegation and context management

- **Slice and report clearly.** Divide work by class type or another clear boundary, and use a fresh subagent per slice when practical. Each worker should verify its slice with line counts and structure read-backs, then report per-method line counts; the parent cross-checks the slices and owns the overall goal.
- **Keep the parent's context slim.** A forked subagent inherits the full parent conversation, and a large parent context can leave little room for the work. Prefer a fresh, non-forked subagent for relaunches. Use marker-string searches instead of full per-method read-backs, and append progress to a file per batch.
- **Don't ping busy workers.** A status message can interrupt an in-flight staging call and force the worker to re-stage its batch. Heavy-body generation can take minutes per turn. Rely on completion notices; probe progress read-only (for example, search for a unique per-class marker). Ping only at a step boundary when there is a real concern.
- **Interpret status and progress together.** A “was stopped before it finished” notice settles one run; it does not establish that the worker is dead. Treat a worker as dead only when its status is `ready` and progress is 0. Confirm no worker is still active before relaunching, or duplicate work may start on the same slice.
- **Make relaunches idempotent.** Probe per-item marker strings before repeating work, and build the item list from a ground-truth class listing in the library, never from memory. A class-level `get_details` listing is a reliable source; memory-derived class lists can be wrong.
