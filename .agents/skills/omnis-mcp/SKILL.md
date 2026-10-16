---
name: omnis-mcp
description: Use when operating Omnis Studio libraries through the Omnis MCP server (mcp__omnis__* tools).
---

# Omnis MCP (Omnis Studio bridge)

Quirks of the `mcp__omnis__` tool surface and the platform behaviour it exposes. The live Studio library, read back via MCP, is the only trustworthy source.

## Golden rules

1. **Omnis fails silently** — bad refs, missing/mismatched columns, wrong notation all return empty, never errors.
2. **Served schemas under-report accepted values** — treat enums as unverified; probe before assuming rejection.
3. **Preview tokens expire quickly and can't be renewed.** — Use them ASAP.

## Preview/apply (methods)

- `preview_set_method` stages (returns token + `methodDiff` + `expires`) → `apply_set_method(token)` commits.
- `expires` is authoritative; lifetime ~2 min (30–60 s seen). Expired apply → `No pending operation found for token`. **Recovery: re-preview the lost methods verbatim** — idempotent for unapplied new methods; whole batches have recovered this way.
- **`variables[]`: every entry needs `"action":"add"`** — omitting it (even for parameters) yields cryptic `{"errors":[""]}`. **Omit `variables` entirely when re-staging a method whose parameters exist** (re-declaration risks duplicates on apply).
- **`methodDiff`** can transiently be empty for method-line changes — when empty, do not assume nothing changed; build the before/after from a `get_details` read of the current method.
- **Bulk:** the bridge handles 12 previews + 12 applies and 10 `create_class` per turn; turn N applies turn N−1's tokens and previews turn N's. One call in a parallel batch can fail with `Error: fetch failed` — retry that call verbatim (no half-state).
- **Context cost:** each preview echoes the full diff back — hundreds of large rewrites exhaust a session's context; split into subagent slices (see `references/subagents.md`).
- No restart/reload or rebuild needed: new lines can be executed immediately; `.omh`/`.json` exports are lagging snapshots of the `.lbs`.

## Served schemas under-report

| Field (`alter_variables` / `preview_set_method.variables`) | Schema enum | Server actually accepts |
|---|---|---|
| `variableScope` | `["parameter"]` | `parameter`, `instance`, `local` (description claims `class`/`task` — verify) |
| `type` | `["character"]` | `character`, `number`, `integer`, `boolean`, `date`, `binary`, `itemref`, `object` |
| `action` (also `alter_constants`, `alter_schema_columns`) | `["add"]` | only `add` verified; description claims update/delete — probe first |

- **`fieldref`/`objref` parameter types are rejected in every spelling tried** — declare the parameter **`itemref`** instead (field/object refs are itemref subtypes).
- **The enums contradict the tools' own description text** — never assume a value is rejected just because the schema omits it.

## Silent type coercion (variables — not constants)

Silent normalization at create time and via `alter_variables`:

| Requested | Becomes | Impact |
|---|---|---|
| integer `short` | `32bit` | cosmetic |
| number `0dp` | `float` | no observed difference |
| date `datetime` | `date2000` | **can change behaviour** — datetimes truncate to date; set the subtype in Studio if time precision matters |

- **Constants do not coerce** (`subtype:"short"` and `type:"date"` echo back unchanged).
- **Local variable names are case-insensitive** (`dDX`/`dDx` merge) — avoid near-duplicate names.

## Reading and inspection quirks

- **Object-level methods:** the slash form (`method="btnX/$event"`) is rejected → `method="$event"` + separate `object="btnX"`.
- **`edit_class` cannot address the library class itself** → set prefs at runtime in `Startup_Task.$construct`: `Do $clib.$prefs.$mouseevents.$assign(kTrue)` (same for `$rmouseevents`/`$keyevents`).
- **Folders:** Create a folder with `create_class` using `classType:"folder"`. Place a class in it via the `folder` param (unique folder *name* only, no `:`-path).
- **Filter before counting:** fresh libraries auto-contain ~19 `#...` systemtable classes + `Startup_Task`; folders are listed alongside classes.
- `preview_set_method` always appends to the end of the class's method list, and there's no tool that can move an existing method.
- **`alter_constants`:** `add` on an existing constant overwrites it in place**. So to change a constant, use add with the same name; add with a new name creates one.
- **`search_library` has index/timing lag** (fresh probes return 0 for just-applied classes) — not a stall detector.
- **`search_docs` (RAG) can time out at 300 s** → `lookup_symbols` (fast, authoritative; batch symbols; `detail:"full"`; qualify ambiguous members) + `omnis://docs` resources.

## Bridge, host, and environment

- The server can run on a separate machine.
- Omnis is single threaded; none of the omnis-mcp tools work while the Omnis process is doing work or is stuck.
- **Omnis is interpreted** — the `.lbs` is the running code; lines written mid-debug (incl. `Breakpoint`) are able to be executed immediately, no restart/reload needed; `.omh`/`.json` exports can be stale since that's a snapshot.
- The MCP offers no tools to control the UI.

## Delegating to subagents

**Read `references/subagents.md` (in this skill's `references/` folder) before fanning Omnis MCP work out to subagents or performing bulk staged changes**.
