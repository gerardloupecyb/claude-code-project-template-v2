---
date: 2026-03-14
topic: lesson-retention-system
---

# Lesson Retention System — 3-Layer Cache

## What We're Building

A system to prevent lessons learned from being lost between sessions. Currently the flywheel (docs/solutions/ + CARL + Supermemory) exists on paper but leaks at every link: lessons aren't captured (compound workflow too heavy), aren't consulted (agent search too costly), and aren't actionable (too vague).

The solution is a 3+1 layer cache system where each layer adds detail, and Claude descends only when needed:

| Layer | Content | Access | Size |
|-------|---------|--------|------|
| CARL rules | One-liner rule | Auto-injected every prompt | ~15-20 rules max |
| LESSONS.md | When/Do/Because | Auto-read every session (in CLAUDE.md) | Cap 50 entries |
| Supermemory project | Structured summary + tags | `recall` at planning time | Unlimited |
| docs/solutions/ | Full pattern + code + anti-patterns | Agent search (fallback or depth) | Unlimited |

## Why This Approach

**Problem diagnosed:** The flywheel leaks at 3 points:
1. **Capture** — `/workflows:compound` is 5 steps, too heavy for "I found something"
2. **Consultation** — `docs/solutions/` requires an agent search (tokens + time, often skipped)
3. **Actionability** — Supermemory/CARL entries too vague, Claude doesn't know when to apply

**Approaches considered:**
- A) LESSONS.md always loaded — solves consultation + actionability
- B) CARL always-on rules as lessons — solves actionability, but limited to ~20 rules
- C) Post-commit hook — solves capture, but user preferred confirmation prompts (b) over auto-hooks

**Chosen:** A + B combined, with Supermemory per-project as primary archive and docs/solutions/ as local git backup.

## Key Decisions

- **LESSONS.md cap: 50 entries** — enough margin before migration, compact enough to read fast
- **Supermemory per-project replaces docs/solutions/ as primary archive** — `recall` is faster than agent search on local files
- **docs/solutions/ becomes write-only backup** — read only when Supermemory unavailable OR need more detail (code examples, full context)
- **Friction level: one confirmation prompt (oui/non)** — no heavy workflow, no auto-hooks
- **CARL RULE_1 updated** — consultation order changes from "check docs/solutions/" to "check LESSONS.md (loaded) → recall Supermemory → agent search docs/solutions/ if needed"
- **`/lesson` skill** — captures in ~10 seconds, proposes CARL promotion if critical/repeated

## Lifecycle of a Lesson

```
Fix non-trivial
    │
    ▼
/lesson → LESSONS.md (quand/faire/parce que, oui/non confirmation)
    │
    ├── if critical or repeated (>=3) → propose CARL rule promotion
    │
    └── if cap 50 reached → migrate oldest 10:
           ├── memory → Supermemory (project tag)
           └── copy → docs/solutions/ (local backup)
           Both in parallel, one decision point.
```

## Changes to Template

| File | Action |
|------|--------|
| `LESSONS.md.template` | NEW — compact file with header, format spec, empty |
| `.claude/skills/lesson/SKILL.md` | NEW — rapid capture skill |
| `CLAUDE.md.template` | MODIFY — add LESSONS.md to file table, update consultation rule, update workflow |
| `init-project.sh` | MODIFY — generate LESSONS.md from template |
| `.carl/domain.template` | MODIFY — update RULE_1 consultation order |
| `docs/solutions/` | NO CHANGE — stays as git-versioned backup |
| Supermemory | NO CHANGE to MCP — just use per-project tagging |

## What We're NOT Doing

- No post-commit hook (user prefers explicit confirmation over auto-triggers)
- No removal of docs/solutions/ (stays as backup + depth)
- No change to Supermemory MCP setup (already installed)
- No change to `/workflows:compound` (stays for heavy/detailed patterns, but `/lesson` handles 80% of cases)

## Open Questions

- Should `/lesson` auto-detect domain from file context, or always ask?
- Should migration from LESSONS.md propose which entries to archive, or oldest-first by default?

## Next Steps

→ Plan implementation with `/gsd:plan-phase` or manual planning
