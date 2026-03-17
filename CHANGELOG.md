# Changelog

Toutes les modifications notables de ce projet sont documentees dans ce fichier.
Format base sur [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
versioning selon [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.0.0] — 2026-03-16

Release majeure — systeme de retention complet, hooks, quality score.

### Added

#### Systeme de retention 4 couches
- `LESSONS.md` — cache chaud des lecons (cap 50, format quand/faire/parce que)
- `DECISIONS.md` — registre ADR-light des decisions (cap ~25, tags, champ Rejete)
- `memory/MEMORY.md` — etat courant de session (lu en premier, mis a jour en fin)
- Integration Supermemory pour archivage cross-projet

#### 7 skills inclus
- `/lesson` — capture rapide de lecons en 10 secondes
- `/session-gate` — validation mecanique de MEMORY.md (13 checks)
- `/context-checkpoint` — sauvegarde rapide avant coupure de session
- `/project-bootstrap` — injection cross-projet de lecons depuis Supermemory
- `/pre-flight` — review multi-agent des plans (4 agents paralleles)
- `/context-manager` — reference de gestion du contexte
- `/project-sync` — synchronisation avec outils externes

#### Hooks anti-compaction
- `pre-compact.sh` — snapshot MEMORY.md avant compaction du contexte
- `session-start.sh` — re-injection automatique de MEMORY.md + LESSONS.md au demarrage

#### Rules et discipline
- `.claude/rules/tool-routing.md` — table de routing des outils + anti-patterns + discipline MCP
- `.claude/rules/flywheel-workflow.md` — workflow de capitalisation
- CARL RULE_0-8 : flywheel, credentials, closure, deviation, tool routing, MCP discipline, planning recall

#### Closure protocol (7 etapes)
- Quality score mecanique (CLEAN/ROUGH)
- Capture automatique des decisions dans DECISIONS.md

#### Documentation
- `docs/GUIDE.md` — guide complet (14 sections, architecture, workflows, FAQ)
- `README.md` — quick start + structure

#### Init script
- `init-project.sh` — genere un projet complet avec tous les templates, skills, hooks, rules, et CARL
