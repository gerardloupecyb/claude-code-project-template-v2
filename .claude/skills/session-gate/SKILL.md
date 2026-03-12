---
name: session-gate
description: "Validation mécanique de l'état de session (MEMORY.md). This skill should be used when validating session state at boundaries. Triggers on: session gate, valider la session, vérifier mémoire, état de la session. Also invoked explicitly with /session-gate."
---

# Session Gate — Validation mécanique de MEMORY.md

Verify that memory/MEMORY.md follows the template rules mechanically.
Read-only, advisory, stateless. Never modify files. Never block the session.

---

## Invocation

| Command | Mode | Checks |
|---|---|---|
| `/session-gate start` | START | 1, 2, 3, 4, 5 |
| `/session-gate end` | END | 1, 4, 5, 6, 7 |
| `/session-gate` (no arg) | BOTH | All 7 |

---

## Pre-check: merge conflicts

Before running any check, scan memory/MEMORY.md for lines starting with
`<<<<<<<`, `=======`, or `>>>>>>>`. If found, report:

```
Session Gate — BLOCKED

  MEMORY.md contains unresolved merge conflicts. Resolve them before continuing.
```

Skip all remaining checks.

---

## The 7 Checks

Run each applicable check. Use `Read` and `Grep` tools on memory/MEMORY.md
and `Bash` for git status. All checks are mechanical — no semantic judgment.

### Check 1 — MEMORY.md exists and is non-empty (START, END)

Read memory/MEMORY.md. Verify:
- File exists
- File contains at least one markdown heading (`^#`)

If missing or empty: `[!!] MEMORY.md missing or empty — create from template`

### Check 2 — Last session age (START) — informational

Find the line matching (case-insensitive) `Dernière session`. Extract the
YYYY-MM-DD date. Calculate days since that date.

Display: `[--] "Dernière session" YYYY-MM-DD (N days ago)`

If date is not parsable as YYYY-MM-DD: `[!!] "Dernière session" date not parsable`

### Check 3 — Deviations cleared (START)

Find the section matching (case-insensitive) `Déviations d'exécution`.
In that section, find lines containing `|`. Skip the first 2 such lines
(table header + separator row). Count remaining `|`-lines as deviation entries.

If count > 0: `[!!] "Déviations d'exécution" has N entries — clear them`
If count == 0: `[ok] "Déviations d'exécution" cleared`

### Check 4 — "Ce qui a été fait" capped at 5 (START, END)

Find the section matching (case-insensitive) `Ce qui a été fait`.
Count `###` headings within that section (stop at the next `## ` heading).

If count > 5: `[!!] "Ce qui a été fait" has N entries (max 5) — archive older ones`
If count <= 5: `[ok] "Ce qui a été fait": N/5`

### Check 5 — "Prochaine étape" present (START, END)

Find the line matching (case-insensitive) `Prochaine étape`.
Verify it exists and does not contain `{{` (template placeholder).

If missing or placeholder: `[!!] "Prochaine étape" missing or still a placeholder`
If present: `[ok] "Prochaine étape" present`

### Check 6 — "Dernière session" is today (END)

Find the line matching (case-insensitive) `Dernière session`. Extract the
YYYY-MM-DD date. Compare to today's date.

If not today: `[!!] "Dernière session" is YYYY-MM-DD, not today — update it`
If today: `[ok] "Dernière session" is today`

### Check 7 — MEMORY.md staged with code (END)

Run `git status --porcelain`. Check:
- If no files are staged at all: skip this check (not applicable)
- If files are staged but memory/MEMORY.md is NOT staged:
  `[!!] Files are staged but MEMORY.md is not — stage it with the code`
- If MEMORY.md is staged: `[ok] MEMORY.md staged with code`

---

## Output format

```
Session Gate — {MODE}

  [ok]  MEMORY.md exists and is non-empty
  [--]  "Dernière session" 2026-03-10 (1 day ago)
  [!!]  "Déviations d'exécution" has 2 entries — clear them
  [ok]  "Ce qui a été fait": 3/5
  [ok]  "Prochaine étape" present

  1 issue found. Fix before continuing.
```

Legend: `[ok]` = pass, `[!!]` = action required, `[--]` = informational.

If any `[!!]` checks exist, report count and recommend fixing.
If no `[!!]` checks, end with: "All clear."

---

## What this skill does NOT do

- Modify any file (read-only, like pre-flight)
- Block the session (advisory only — user decides)
- Judge content quality ("is this next step good?")
- Read STATE.md (no coupling with GSD)
- Require state between invocations (stateless)
