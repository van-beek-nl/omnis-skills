---
name: omnis
description: Use when doing Omnis programming — writing, modifying, reviewing, or debugging Omnis code or working inside a .lbs Omnis library.
---

## Pitfalls

Consolidated pitfalls index; the sections below add only what the table doesn't.

| You might write or assume | Omnis reality |
|---|---|
| `x = expr` | `Calculate x as expr` |
| `Return value` | `Quit method value`; caller: `Do method name Returns var` |
| `s1 + s2` for character values | `con(s1, s2)` — `+` is numeric only |
| `val()`, `round()`, `random()`, `year()` | don't exist — coerce `s * 1.0`; randoms are `rand`/`randintrng`/`randrealrng` |
| `pick(cond, ifTrue, ifFalse)` | zero-based: `pick(n, v0, v1, …)`; `pick(bool, A, B)` returns **A when FALSE** |
| `Calculate lvRef as $win.$open()` | the reference comes back only via `Do $win.$open() Returns lvRef` |
| `cOther.iVar` (cross-class variable access) | rejected — getter/setter methods per class |
| `$fillcolor`, `$caption`, `$maxwidth`/`$maxheight` | `$backcolor`, `$title`, `$minwidth`/`$minheight` |
| `dDX` and `dDx` are distinct variables | names are case-insensitive |
| a bound entry field repaints after its variable changes in code | call `Do $cinst.$objs.fValue.$redraw()` |
| list rows before columns; OJSON flattens a JSON array of objects into columns | define columns first (else empty rows); OJSON packs each object into a row-typed `C1` column |
| the column names of a table live on its table class | they live on the schema class |
| `Do lvList.$add(...)` before defining list columns | define columns first, e.g. `Do lvList.$cols.$add('role',kCharacter,kSimplechar)`; otherwise rows are empty |

## Public vs private methods

A leading `$` makes a method **public** — the `$` is part of the name and must appear on every call site. `$cinst` = current instance (the `self` equivalent; `$cfield`/`$ctask` likewise).

**Public (`$name`)** — callable on any instance via notation, `$` + name + parens:

- `Do oWorker.$doWork(2)`
- `Calculate lvInt as oCalc.$add(2,2)`
- `Do $cinst.$doWork(2)`

Return value: `Quit method <expr>`. Properties (not methods) take no parens: `Do oWorker.$state.$assign(v)`.

**Private (no `$`)** — not protected; reachable only from the class's own code, and only via:

- `Do method <name> (params) Returns <var>` — parameter list and `Returns <var>` are both optional.

`Do method` resolves in the current class only (field/line methods first, then class methods) — it cannot reach another instance or class. Notation calls (`Do <instance>.<name>(…)`, `Calculate … as <instance>.<name>(…)`) accept `$`-prefixed names only. So: defining `$name` ⇒ call it as `$name`, never `oWorker.name(…)`; a no-`$` method is unreachable outside its own class.

Note: a small set of names is reserved and must not be used for custom methods/properties (`$add`, `$assign`, `$canassign`, `$chaincount`, `$default`, `$findname`, `$ident`, `$isinherited`, `$makelist`, `$name`, `$ref`, `$serialize`, `$wind`).

## Language notes

**Strings/numbers.** `con()` coerces non-strings (`con('May ', 8, 'th')` → `'May 8th'`). A `+` between characters can pass the method validator uncaught — always treat it as a bug. `format(format, variable)` takes format **first**: `format('0D##########', nValue)`. Zero-pad fractional parts when hand-building fixed-decimal output (bug: `"160.0"` vs `"160.00"`). `Do … Returns <itemrefVar>` sets the reference but does not assign to it — assign via `Calculate` or `Do <itemref>.$assign(value)`.

**Functions.** Verified: `getdatetime()`, `format()`, `left()`, `right()`, `mid()`, `pos()`, `upp()`, `low()`, `len()`, `trim()`, `int()`, `con()`, `pick()`, `rand()`, `randintrng()`, `randrealrng()`. No date literals — date math in integer seconds from `getdatetime()`. `pos()`/`mid()` are **one-based** (porting hazard). Verify unfamiliar names against the Omnis documentation for the target version.

**`pick()`.** A boolean maps `kTrue`→1, `kFalse`→0, so it fails silently in *both* branches; flipping the condition does not fix it — value order must match the produced index:

```omnis
# name absent -> len=0 true (1) -> lvAddress; present -> false (0) -> block
Calculate lvText as pick(len(lvName)=0, <name block>, lvAddress)
```

**Loops/branching.** `While` tests at the start; `Repeat`/`Until` tests at the end and always runs at least once. `If <calc>`/`Else`/`Else If`/`End If`: non-zero is true, zero is false, body runs when true.

**Naming.** In Omnis, variables are often prefixed with a letter indicating their scope to avoid naming clashes within a class. Follow the codebase's naming conventions.

## Notation notes

- Correct vs wrong: `Do oWorker.$compute(sInput) Returns nTotal` vs `Do oWorker.compute(sInput) Returns nTotal` → "Problem with notation". Same-class calls keep plain names.
- Correct vs wrong: `Do $windows.wPickDialog.$open() Returns lvWinRef` vs `Calculate lvWinRef as $windows.wPickDialog.$open()` → empty reference, the window still opens.
- Dynamic bracket names work (`$cinst.$objs.[cName].$picture.$assign(…)`); `.$assign` can report success via `Returns lFlag`.
- **"Problem with notation"** causes: (1) an entry-field object named like an instance variable makes bare notations ambiguous — rename the *ivar* (type prefix) and rebind the field's `$dataname` (`Queue set current field fField` still targets the field object); (2) a child-object method missing its `$` prefix; (3) bare component names.
- `Ok message 'Title' {cVar}` — braces around the message variable are required; `[kCr]` for line breaks.

## lists

- `list.$add(colVal, …)` inserts a line and returns an itemref; define columns first:
```omnis
Do lvEntries.$cols.$add('role', kCharacter, kSimplechar)
Do lvEntries.$cols.$add('row_identifier', kCharacter, kSimplechar)
Do lvEntries.$add('a', 'first row')
```

## OJSON
- **`OJSON.$jsontolistorrow` on a JSON array of objects packs each object into a single row-typed column `C1`** — members one level down (`lvEntries.C1.name`); docs imply flattening, it does not.

## Quirks
- Under the hood, a datetime is stored as an integer scaled to its format's finest unit of precision; subtracting two datetimes subtracts these raw integers without normalizing mismatched precisions.
- **Omnis fails silently**: bad references, missing/mismatched list columns, wrong indexes, and truncated types all yield *empty values, never errors*

## Library, runtime, verification

- Every class type can contain methods (object, schema, table, search, code, window, task, userconstants); object/table classes also have `$construct`/`$destruct`.
- **Large text trees**: a whole-tree `rg --files` glob can be too slow — use `find` or a scoped `rg`.
