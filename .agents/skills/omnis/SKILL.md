---
name: omnis
description: Use when doing Omnis programming — writing, modifying, reviewing, or debugging Omnis code (methods, notation, lists, entry fields, timers, events, OJSON) or working inside a .lbs Omnis library. Pitfalls, silent-failure assumptions, platform quirks.
---

# Omnis Programming

**Omnis is silent by default**: bad references, missing/mismatched list columns, wrong indexes, and truncated types all yield *empty values, never errors* — verify intermediates deliberately. Agent-workflow notes (bulk work, delegation, staged-change tooling) live in the tool-specific supplement [workflow.md](references/workflow.md); this file is pure language reference.

## 1. Quick reference

Consolidated pitfalls index; the sections below add only what the table doesn't.

| You might write or assume | Omnis reality |
|---|---|
| `x = expr` | `Calculate x as expr` |
| `Return value` | `Quit method value`; caller: `Do method name Returns var` |
| `s1 + s2` for character values | `con(s1, s2)` — `+` is numeric only |
| `val()`, `round()`, `random()`, `year()` | don't exist — coerce `s * 1.0`; randoms are `rand`/`randintrng`/`randrealrng` |
| `pick(cond, ifTrue, ifFalse)` | zero-based: `pick(n, v0, v1, …)`; `pick(bool, A, B)` returns **A when FALSE** |
| `Do $windows.$display(wWin)` | `Open window instance wWin` |
| `Calculate lvRef as $win.$open()` | the reference comes back only via `Do $win.$open() Returns lvRef` |
| method calls with parens / bare component names / `oWorker.method` | no parens, no bare names, child-object methods need `$`: `Do method tick`, `Do $cinst.$objs.oTimer.$resettimer`, `Do oWorker.$method(args)`; properties via `.$assign(v)` |
| `cOther.iVar` (cross-class variable access) | rejected — getter/setter methods per class |
| `$fillcolor`, `$caption`, `$maxwidth`/`$maxheight` | `$backcolor`, `$title`, `$minwidth`/`$minheight` |
| `dDX` and `dDx` are distinct variables | names are case-insensitive |
| datetime subtraction → milliseconds | returns seconds |
| a bound entry field repaints after its variable changes in code | call `Do $cinst.$objs.fValue.$redraw()` |
| entry-field `$text`/`$calculation`/`$formatmode`/`$formatstring` are settable | they are not — workarounds in §5 |
| list rows before columns; OJSON flattens a JSON array of objects into columns | define columns first (else empty rows); OJSON packs each object into a row-typed `C1` column |
| re-adding an existing constant errors | it updates the constant in place |
| code needs recompilation or a restart | interpreted — the `.lbs` library is the code that runs |

## 2. Language notes

**Strings/numbers.** `con()` coerces non-strings (`con('May ', 8, 'th')` → `'May 8th'`). A `+` between characters can pass the method validator uncaught — always treat it as a bug. `format(format, variable)` takes format **first**: `format('0D##########', nValue)`. Zero-pad fractional parts when hand-building fixed-decimal output (bug: `"160.0"` vs `"160.00"`). `Do … Returns <itemrefVar>` sets the reference but does not assign to it — assign via `Calculate` or `Do <itemref>.$assign(value)`.

**Functions.** Verified: `getdatetime()`, `format()`, `left()`, `right()`, `mid()`, `pos()`, `upp()`, `low()`, `len()`, `trim()`, `int()`, `con()`, `pick()`, `rand()`, `randintrng()`, `randrealrng()`. No date literals — date math in integer seconds from `getdatetime()`. `pos()`/`mid()` are **one-based** (porting hazard). Verify unfamiliar names against the Omnis documentation for the target version.

**`pick()`.** A boolean maps `kTrue`→1, `kFalse`→0, so it fails silently in *both* branches; flipping the condition does not fix it — value order must match the produced index:

```omnis
# name absent -> len=0 true (1) -> lvAddress; present -> false (0) -> block
Calculate lvText as pick(len(lvName)=0, <name block>, lvAddress)
```

`If`/`Else`/`End If` is least error-prone for boolean-selected strings.

**Loops/branching.** `While` tests at the start; `Repeat`/`Until` tests at the end and always runs at least once. `If <calc>`/`Else`/`Else If`/`End If`: non-zero is true, zero is false, body runs when true.

**Naming.** In Omnis, variables are often prefixed with a letter indicating their scope to avoid naming clashes within a class. Follow the codebase's naming conventions.

## 3. Notation notes

- Correct vs wrong: `Do oWorker.$compute(sInput) Returns nTotal` vs `Do oWorker.compute(sInput) Returns nTotal` → "Problem with notation". Same-class calls keep plain names.
- Correct vs wrong: `Do $windows.wPickDialog.$open() Returns lvWinRef` vs `Calculate lvWinRef as $windows.wPickDialog.$open()` → empty reference, the window still opens.
- Dynamic bracket names work (`$cinst.$objs.[cName].$picture.$assign(…)`); `.$assign` can report success via `Returns lFlag`.
- **"Problem with notation"** causes: (1) an entry-field object named like an instance variable makes bare notations ambiguous — rename the *ivar* (type prefix) and rebind the field's `$dataname` (`Queue set current field fField` still targets the field object); (2) a child-object method missing its `$` prefix; (3) bare component names.
- `Ok message 'Title' {cVar}` — braces around the message variable are required; `[kCr]` for line breaks.

## 4. Types, lists, OJSON

- Types: character, number, integer, date/datetime, binary, boolean, object/itemref; field and object references are subtypes of itemref.
- Silent subtype coercion on variables (create and update): `shortint`→`32bitint` (cosmetic), `0dp`→`float` (harmless), `datetime`→`date2000` (**time truncated before comparison** — set the subtype to datetime when precision matters). Constants are never coerced (`short` echoed, `type:"date"` accepted).
- Clamp tick deltas to 0.016–0.25 s and ignore <0.008 s (double-event guard).
- `list.$add(colVal, …)` inserts a line and returns an itemref; define columns first:

```omnis
Do lvEntries.$cols.$add('role', kCharacter, kSimplechar)
Do lvEntries.$add('a', 'first row')
```

- OJSON: object members sit at `lvEntries.C1.name` (docs imply flattening — they do not); a nonexistent column reference is a silent empty.
- Param rows for list-display windows: the dialog builds text from the **value** of a column named `C1`:

```omnis
Do lvWinRef.$setDisplayCols(TMObjs.$makeparamrow('C1', 'text'))
```

## 5. Windows and UI

- Titled window: `$style=kTitle` plus `$title`. Some properties are readable but not writable — if one is rejected, find the supported alternative (`$showtitle` rejected → `$style=kTitle`).
- Shape fill: `Do $cinst.$bobjs.oBg.$backcolor.$assign(kBlack)` (named colour constants, not RGB).
- Entry-field workarounds (for the unsettable properties in the table): static captions → background `Label` objects (`bobjs`); dynamic values → bind the field to an instance variable, set in code, format with `format()`; calculated/zero-empty → `$calculated=kTrue` + `$zeroempty=kTrue`; conditional → toggle `$visible` at runtime.
- `$redraw([bSetcontents=kTrue, …])` reloads control data by default — that's why the redraw shows the new value.
- `evKey`: `pEventCode` int, `pKey` **char**, `pSystemKey` int; letters have `pSystemKey=0` — compare via `upp(pKey)='R'`; special keys are exact ints (Esc=32, up=17, down=18).
- Timers: adding one auto-generates an `$event` stub containing `On evTimer` — preserve that line when rewriting the body. Window-level and component timers may both fire → guard against duplicate ticks (clamps above).

## 6. Library, runtime, verification

- Every class type accepts user methods (object, schema, table, search, code, window, task, userconstants); object/table classes also get automatic `$construct`/`$destruct`. Field definitions live on the schema class — if a table's field listing is empty, inspect the schema class.
- Extension `.lbs` — Omnis silently accepts others (e.g. `.oms`); validate at creation (fix: close in Studio, rename on disk). A fresh library contains ~19 `#…` systemtable classes plus `Startup_Task` — filter before counting.
- No restart/reload needed: new lines (including `Breakpoint`) execute immediately; `.omh`/`.json` exports are lagging snapshots of the `.lbs`.
- Library preferences can't be set through the class editor — set them at runtime in `Startup_Task.$construct`: `Do $clib.$prefs.$mouseevents.$assign(kTrue)` (plus `$rmouseevents`, `$keyevents`).
- **Verify**: at a first `Breakpoint`, exercise the whole chain once and inspect every intermediate (lists, references, calculation strings) before any window is expected to render. Do not execute code from the agent — verify statically/read-only, re-verify applied code via read-backs, then hand the user a run checklist (regression, demo, expected outputs).
- **Specs**: check internal consistency before implementing — counts, asset lists, positions can disagree; the data/asset set has been ground truth; flag the discrepancy.
- **Large text trees**: a whole-tree `rg --files` glob can be too slow — use `find` or a scoped `rg`.
