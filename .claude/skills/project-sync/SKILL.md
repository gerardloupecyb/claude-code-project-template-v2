---
name: project-sync
description: "Synchronise l'état projet avec les outils externes (Linear, GSD). Se déclenche sur : project sync, sync project, synchroniser, mettre à jour Linear, sync linear. Aussi invoqué explicitement avec /project-sync."
---

# Project Sync — Synchronisation avec les outils externes

Reads `.claude/integrations.md` and `memory/MEMORY.md`. Advisory with confirmation
before any write. Never executes actions without explicit user confirmation.

---

## Invocation

| Command | Mode | Checks |
|---|---|---|
| `/project-sync` | END | 1, 2, 3, 4 (per active integrations) |
| `/project-sync start` | START | 1 only |
| `/project-sync linear` | END | 1, 2, 3 |
| `/project-sync gsd` | END | 1, 4 |

---

## Pre-check: parse integrations.md

Read `.claude/integrations.md`. Parse `key: value` lines (case-insensitive). Extract:
`linear`, `linear_team_id`, `linear_project_id`, `gsd`, `supermemory`.

If file missing: report `[!!] .claude/integrations.md missing — copy from template` and stop.
If any line contains `{{`: report `[!!] integrations.md has unfilled placeholders` and stop.

---

## Check 1 — integrations.md valid (START, END)

Verify:
1. File exists at `.claude/integrations.md`
2. No lines containing `{{` (unfilled placeholder)
3. If `linear: true` → `linear_team_id` and `linear_project_id` must be non-empty

If check 3 fails: `[!!] linear: true but linear_team_id or linear_project_id is empty`
If all pass: `[ok] integrations.md valid (linear: [value], gsd: [value])`

---

## Check 2 — Linear: tasks to close (END, if `linear: true`)

Read `memory/MEMORY.md` section "Ce qui a été fait" — collect `###` headings.
Call `mcp__linear__list_issues` for open issues in `linear_project_id`.

If MCP unavailable: `[--] Linear MCP unavailable — skipping`

For each `###` heading, search for a matching open issue (keywords overlap).

Report candidates and ask per issue:
```
[--] Issue to close: #XX "Add session gate validation" ← "feat: session-gate"
     Close? (yes/no)
```
If yes: call `mcp__linear__save_issue` with `stateId` = done + comment = heading text.
If no: skip silently.

---

## Check 3 — Linear: next step covered (END, if `linear: true`)

Read `memory/MEMORY.md` "Prochaine étape" value.
Search Linear open issues for title match.

If found: `[ok] "Prochaine étape" covered by #XX — [title]`
If not found:
```
[--] No open issue for: "[prochaine étape text]"
     Create one? (yes/no)
```
If yes: call `mcp__linear__save_issue` with title = prochaine étape, team = `linear_team_id`.

---

## Check 4 — GSD coherence (END, if `gsd: true`)

Read `STATE.md` (if exists). Extract current phase.
Read `memory/MEMORY.md` "Prochaine étape".

Display: `[--] GSD Phase [X.Y] | Prochaine étape: "[text]"` — informational only.
If STATE.md absent: `[--] STATE.md not found — GSD not yet initialized`

---

## Output format

```
Project Sync — END

  [ok]  integrations.md valid (linear: true, gsd: true)

  Linear:
  [--]  Issue to close: #LAI-42 "Add session gate" ← "feat: session-gate"
        → Close? (yes) ✓ Closed
  [--]  No issue for "Implémenter project-sync" → Create? (no) skipped

  GSD:
  [--]  Phase 3.2 | Prochaine étape: "Implémenter project-sync skill"

  Sync complete.
```

Legend: `[ok]` = verified/done, `[!!]` = error blocking, `[--]` = informational/question.

---

## What this skill does NOT do

- Write anything without explicit user confirmation
- Replace Linear as project management tool
- Create issues from "Déviations d'exécution" (v1 scope)
- Sync docs/plans/ to Linear (v1 scope)
- Replace session-gate (different responsibilities)
- Block the session (advisory only)
