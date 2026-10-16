---
name: omnis-mcp
description: "Use when operating Omnis Studio libraries through the Omnis MCP server (mcp__omnis__* tools): creating or editing classes, methods, variables, constants, visual objects, reading library contents, or when an Omnis MCP call fails, misreports, or fails silently."
---

# Omnis MCP (Omnis Studio bridge)

Quirks of the `mcp__omnis__` tool surface and the platform behaviour it exposes. The live Studio library, read back via MCP, is the only trustworthy source.

## Golden rules

1. **Never execute arbitrary code from the agent** — verify statically; leave execution to the user (the guarded test-execution tools are the only exception).
2. **Served schemas under-report accepted values** — treat enums as unverified; probe before assuming rejection.
3. **Preview tokens are short-lived and unrenewable** — apply on the next turn; re-preview is the recovery path.
4. **Omnis fails silently** — bad refs, missing/mismatched columns, wrong notation all return empty, never errors.
5. **Verify every write by read-back**; after a bridge incident, re-establish ground truth before writing.
6. **Read-only probes are safe** and double as health checks (minimal `get_details` with `libraryName="*"`).

## Preview/apply (methods)

`preview_set_method` stages (returns token + `methodDiff` + `expires`) → `apply_set_method(token)` commits.

- `expires` is authoritative; lifetime ~2 min (30–60 s seen). Expired apply → `No pending operation found for token`. **Recovery: re-preview the lost methods verbatim** — idempotent for unapplied new methods; whole batches have recovered this way.
- **`variables[]`: every entry needs `"action":"add"`** — omitting it (even for parameters) yields cryptic `{"errors":[""]}`. **Omit `variables` entirely when re-staging a method whose parameters exist** (re-declaration risks duplicates on apply).
- **`methodDiff`** can transiently be empty for method-line changes — when empty, do not assume nothing changed; build the before/after from a `get_details` read of the current method.
- **Bulk:** the bridge handles 12 previews + 12 applies and 10 `create_class` per turn; turn N applies turn N−1's tokens and previews turn N's. One call in a parallel batch can fail with `Error: fetch failed` — retry that call verbatim (no half-state).
- **Context cost:** each preview echoes the full diff back — hundreds of large rewrites exhaust a session's context; split into subagent slices (see `references/bulk.md`).

## Served schemas under-report

| Field (`alter_variables` / `preview_set_method.variables`) | Schema enum | Server actually accepts |
|---|---|---|
| `variableScope` | `["parameter"]` | `parameter`, `instance`, `local` (description claims `class`/`task` — verify) |
| `type` | `["character"]` | `character`, `number`, `integer`, `boolean`, `date`, `binary`, `itemref`, `object` |
| `action` (also `alter_constants`, `alter_schema_columns`) | `["add"]` | only `add` verified; description claims update/delete — probe first |

- **`fieldref`/`objref` parameter types are rejected in every spelling tried** — declare the parameter **`itemref`** instead (field/object refs are itemref subtypes); proven clean.
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

## Writing method lines

### Wrong → right

| Write | Not | Note |
|---|---|---|
| `Calculate v as expr` | `v = expr` | validator rejects bare assignment |
| `Quit method <value>` | `Return <value>` | no `Return` command; caller uses `Do method <name> Returns <var>` |
| per-class getter/setter methods | cross-class variable access | validator rejects scalars and list dot-refs; no shared state |
| `con(s1, s2, …)` | `s1 + s2` on character operands | `+` is numeric arithmetic; validator does not catch it (`con` also coerces non-strings) |
| `pick(bool, A, B)` — A if FALSE, B if TRUE; or explicit `If`/`Then` | 1-based `pick(condition, ifTrue, ifFalse)` | `pick()` is zero-based; wrong in both branches, no error; flipping the condition does not fix it |

Re-derive types, indexes, and units by hand for nontrivial expressions.

### Call and notation forms

- No trailing `()`: `Do method $tickGame`; same-class calls use plain names.
- **Child-object calls require the `$`-prefixed method name** (plain segment → `Problem with notation`): `Do oCalc.$computeExpression(sExpr) Returns nResult`.
- **Bare component names are rejected** → `$cinst.$objs` / `$cinst.$bobjs`, assignment via `.$assign(value)`; dynamic `[cName]` bracket names work.
- Show a window with `Open window instance wMyWindow` (`Do $windows.$display(...)` rejected).
- **Window refs only via `Do … Returns`** — `Calculate lvRef as $windows.wMyDialog.$open()` opens the window and returns the ref empty, silently.
- **Entry-field object named the same as an instance variable** → ambiguous notations: keep the field name, rename the ivar with a type prefix (`sExpr`, `nResult`), rebind the field's `$dataname`.
- **Bound entry fields do not repaint** on ivar change in code (stale until a click) → `Do $cinst.$objs.fOutput.$redraw()`.
- `Ok message 'Title' {cVar}` — braces around the message variable required; `[kCr]` for line breaks.

### Language and event facts

- No `For <var> from a to b` (the data-driven `For field value` / `For each line in list` exist) → `While`/`End While` or `Repeat`/`Until` (repeat runs at least once); no `random()`/`round()`/`year()`/`val()` → `rand`/`randintrng`/`randrealrng`, `t*1.0`; **datetime subtraction yields seconds**.
- Verified side-effect-free whitelist: `getdatetime`, `format`, `left`, `right`, `mid`, `pos`, `upp`, `low`, `len`, `trim`, `int`; date math = integer seconds on `getdatetime()`, no date literals.
- **`evKey`:** `pEventCode` integer, **`pKey` character**, `pSystemKey` integer (Esc=32, up=17, down=18; letters = 0); letters via `upp(pKey)`, specials as exact ints.
- Adding a Timer auto-generates an `$event` stub containing `On evTimer` — preserve that line when rewriting the body; window-level and component events both can fire → guard against double ticks.
- Shape fill is `$backcolor` (not `$fillcolor`), named colour constants; window title is `$title` (`$caption` read-only); `$maxwidth`/`$maxheight` invalid, `$minwidth`/`$minheight` work.
- All class types accept user methods; object/table additionally get automatic `$construct`/`$destruct`.

## Silent runtime failures

Plumbing is validated silently — every one of these "just came back empty" with no error:

- A nonexistent list-column reference returns empty (no error).
- **List `$add` before columns are defined → empty rows** — define columns first: `Do lvList.$cols.$add('role',kCharacter,kSimplechar)`.
- **`OJSON.$jsontolistorrow` on a JSON array of objects packs each object into a single row-typed column `C1`** — members one level down (`lvEntries.C1.name`); docs imply flattening, it does not.
- **Param-row magic:** some dialog plumbing reads the **value of a column literally named `C1`** — `$makeparamrow('text','')` silently yields blank text; pass `Do lvRows.$makeparamrow('C1','text')` (value = the list column name). When in doubt, **read the consuming method**.

**Debugging rule:** exercise the whole chain once at a first `Breakpoint` (user-run), inspecting every intermediate value, before any window is expected to render.

## Reading and inspection quirks

- **Table classes return no objects** — read fields from the **schema class**.
- **Object-level methods:** the slash form (`method="btnX/$event"`) is rejected → `method="$event"` + separate `object="btnX"`.
- **Readable ≠ writable:** `$showtitle` is returned but rejected when set → use `$style=kTitle` + `$title`.
- **`edit_class` cannot address the library class itself** → set prefs at runtime in `Startup_Task.$construct`: `Do $clib.$prefs.$mouseevents.$assign(kTrue)` (same for `$rmouseevents`/`$keyevents`).
- **Entry-field `$text`/`$calculation`/`$formatmode`/`$formatstring` are NOT settable via `edit_visual_objects`** → background `Label`s (bobjs; `$text` settable) for captions + bound entry fields set in code (`format('0D##########', nValue)`, `$calculated=kTrue`, `$zeroempty=kTrue`).
- **Folders:** `create_class` with `classType:"folder"`; referenced via the `folder` param.
- **Filter before counting:** fresh libraries auto-contain ~19 `#...` systemtable classes + `Startup_Task`; the listing includes folders.
- **No method-reorder API** — new methods append (cosmetic).
- **`alter_constants` `add` on an existing constant updates in place** (idempotent, not an error) — safe for re-tuning.
- **`search_library` has index/timing lag** (fresh probes return 0 for just-applied classes) — not a stall detector.
- **`search_docs` (RAG) can time out at 300 s** → `lookup_symbols` (fast, authoritative; batch symbols; `detail:"full"`; qualify ambiguous members) + `omnis://docs` resources.

## Bridge, host, and environment

- The server can run on a **separate machine**; `create_library` "locked or in use and cannot be deleted" = the host cannot see/write that path, not a lock (recorded setup: only `C:\Users\Public` writable).
- **Bad Studio state stalls even read-only calls** → restart Studio → minimal read health check → resume.
- **Transport drops mid-session** (`http/request failed`) → re-verify the library's state before writing.
- **Non-`.lbs` extensions are silently accepted** → use `.lbs`; fix = close in Studio, rename on disk.
- **Omnis is interpreted** — the `.lbs` is the running code; lines written mid-debug (incl. `Breakpoint`) execute immediately, no restart ever; `.omh`/`.json` exports can lag — when in doubt, re-read live.

## Final verification

- **No UI-driving / task-execution tool exists** — read back everything applied (classes, methods, lines), then hand the user a concrete checklist: what to run, what output to expect.
- **A test class that was created and executed crashed the MCP server** (suspected bad syntax) — `create_test`/`execute_test` (guarded executor: per-test timeouts, teardown) is the only sanctioned agent-side execution path: short, statically verified tests, one class at a time.

## Delegating to subagents

**Read `references/subagents.md` (in this skill's `references/` folder) before fanning Omnis MCP work out to subagents or performing bulk staged changes**.
