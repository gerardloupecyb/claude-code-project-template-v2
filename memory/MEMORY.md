# MEMORY.md — project-template-v2

> Fichier d'état courant. Lu en premier à chaque session.
> Mis à jour en fin de session avant le commit.

---

## État du projet

**Statut :** [ ] En démarrage  [x] En cours  [ ] Bloqué  [ ] Terminé
**Dernière session :** 2026-03-20
**Prochaine étape :** Valider sync upstream en conditions réelles (tester parcours immédiat /gsd:fast, vérifier que plan depth et execution posture fonctionnent avec ce:plan-beta v2.46)

---

## Contexte courant

Template de projet Claude Code avec skills personnalisés (session-gate, context-manager, project-sync, pre-flight) et intégrations CARL/GSD.

---

## Ce qui a été fait

### 2026-03-20 — Upstream sync CE v2.46 + GSD v1.27 — DONE

- Analyse des écarts (plan deepened avec 3 agents, 21→10 actions, 7 coupées)
- Phase 0 : DECISIONS.md créé (DEC-001 auto memory, DEC-002 pre-compact.sh), tag rollback
- Phase 1 : CE v2.40→v2.46 (GitHub, features: execution posture, auto memory, technical design), GSD v1.22.4→v1.27.0 (npm)
- Phase 2 : 8 edits template — parcours immédiat, plan depth, regression gate, auto memory, autonomous/review conditionnel, init-project.sh, flywheel 1.5, README integrations
- CLAUDE.md.template 465 lignes (+18), 5 fichiers modifiés, +47 lignes total

### 2026-03-16 — Execution quality hybridation (ce-work x GSD) — DONE

- Analyse comparative gsd:execute-phase vs ce-work (forces/faiblesses)
- Décision : hybridation via .claude/rules/ plutôt que remplacement
- Créé `.claude/rules/execution-quality.md` (42 lignes, 630 tokens) — 3 patterns : system-wide test check, post-deploy monitoring, commit heuristics
- CLAUDE.md.template mis à jour (outils actifs + phase exécution)
- Released v2.1.0

### 2026-03-16 — Context management improvements (10/10 DONE)

- Plan créé + deepened avec 7 agents parallèles (14 gaps identifiés, 12 corrections intégrées)
- Branch `feat/context-management-improvements`, 7 commits incrémentaux
- Changes 1-5, 8-9: rules, COT, skills, session-gate checks 9-10, CARL RULE_6/7
- Change 6: hooks pre-compact.sh + session-start.sh + settings.json + init-project.sh updated
- Change 10: GLOBAL_RULE_9 step 6 (MCP discipline auto-update) dans ~/.carl/global + blueprint
- Validation: test project généré, toutes les AC vérifiées, cleanup OK

### 2026-03-13 — Initialisation

- Structure projet créée depuis project-template/
- Mise à jour du plugin compound-engineering (v2.40.0 — 47 skills, 28 agents)
- Création du MEMORY.md depuis le template

---

## Décisions actives

| Décision | Raison | Date |
|----------|--------|------|
| CARL RULE_6/7 (pas 8/9) | Dernier actif = RULE_5, séquentiel | 2026-03-16 |
| Check 9 cible docs/plans/ (pas PLAN.md) | PLAN.md n'existe pas dans le template | 2026-03-16 |
| trap exit 0 (pas set -euo pipefail) | Hooks doivent toujours exit 0 | 2026-03-16 |
| 4 matchers SessionStart | resume + clear ajoutés en plus de startup + compact | 2026-03-16 |
| git add scopé MEMORY+LESSONS only | Eviter staging de fichiers en cours d'écriture | 2026-03-16 |
| Flywheel extrait vers .claude/rules/ | CLAUDE.md 505 lignes dépasse le budget 200 d'Anthropic | 2026-03-16 |
| Quality checks CE via rules, pas remplacement GSD | Hybridation : GSD orchestration + ce-work quality patterns injectés via .claude/rules/execution-quality.md | 2026-03-16 |
| DEC-001 MEMORY.md source de vérité, auto memory read-only | CE v2.45 scanne auto memory mais hooks ne gèrent que MEMORY.md | 2026-03-20 |
| DEC-002 Garder pre-compact.sh, pas GSD context monitor | Risque double-snapshot, ordering indéterminé | 2026-03-20 |
| CE v2.46 via GitHub (pas npm v2.42) | Features requises (posture, auto memory, tech design) absentes de npm | 2026-03-20 |

---

## Blocages et questions ouvertes

- [ ] Aucun blocage actuel

---

## Déviations d'exécution

> Vidé en début de session suivante. Max 5 entrées.
> Si la table atteint 5 entrées, signaler que le plan nécessite révision.

| Étape prévue | Action réelle | Raison | Date |
|---|---|---|---|

---

## Patterns découverts cette semaine

Pattern découvert : injecter des quality patterns cross-workflow via `.claude/rules/` plutôt que modifier les fichiers source d'un workflow externe (GSD). Le mécanisme rules/ est auto-chargé par tous les subagents.
→ Potentiellement cross-projet si validé en usage réel.

---

## Stack et config

- Claude Code avec CARL (context-aware rules)
- GSD workflow system
- Compound Engineering Plugin v2.40.0
- Variables d'environnement : voir `.env`

---

## Liens utiles

- Projet local : /Users/gerardvinou/Claude code/Claude Projects/project-template-v2

<!-- pre-compact snapshot -->
**Snapshot pre-compaction** (2026-03-20 13:11)

- **Prochaine etape:** ** Implémenter plan Layer 2 retention gaps (DECISIONS.md, closure quality score, /project-bootstrap, session-gate checks 11-13, CARL RULE_8) — voir docs/plans/2026-03-16-002
- **Derniers commits:**
  - a0e28e8 pre-compact snapshot
  - 824b941 docs: update MEMORY.md — session end, execution quality hybridation done
  - 22b35de feat: inject ce-work quality patterns into GSD executor via rules
- **Fichiers modifies:**
  - ?? DECISIONS.md
  - ?? docs/analyses/
  - ?? docs/plans/2026-03-20-001-analysis-upstream-sync-gaps-plan.md
<!-- /pre-compact snapshot -->
