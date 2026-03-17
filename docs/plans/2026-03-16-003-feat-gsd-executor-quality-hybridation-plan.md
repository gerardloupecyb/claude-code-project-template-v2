---
title: "feat: Inject ce-work quality patterns into GSD executor via rules"
type: feat
status: completed
date: 2026-03-16
---

# Inject ce-work Quality Patterns into GSD Executor

## Overview

Enrichir les subagents gsd-executor avec les quality patterns de `compound-engineering:ce-work` (system-wide test check, post-deploy monitoring, incremental commit heuristics) sans modifier les fichiers GSD core. Le vecteur d'injection est un fichier `.claude/rules/execution-quality.md` qui est auto-chargé par Claude Code dans toute session ou subagent du projet.

## Problem Statement

Le workflow actuel GSD planifie bien (`gsd:plan-phase`) et vérifie bien (gsd-verifier → VERIFICATION.md), mais l'exécution manque de quality checks granulaires au niveau du code :

1. **Pas de system-wide test check** — le gsd-executor commit sans vérifier les callbacks, middleware, observers, state lifecycle, ou interface parity
2. **Pas de post-deploy monitoring** — aucune obligation de documenter quoi surveiller après déploiement
3. **Commit heuristics basiques** — GSD commit par task atomiquement, mais sans évaluer si le changement est complet et testable

`ce-work` a ces trois patterns, mais remplacer `gsd:execute-phase` par `ce-work` casserait le state tracking, les waves parallèles, la vérification, et l'auto-advance.

## Proposed Solution

Créer `.claude/rules/execution-quality.md` — un fichier unique de **<60 lignes** qui injecte les quality checks comme directives. Le mécanisme `.claude/rules/` garantit que tous les subagents (y compris gsd-executor) chargent ces règles automatiquement.

## Technical Approach

### Architecture

```
┌─────────────────────────┐
│   gsd:execute-phase     │  ← orchestrateur (waves, state, verification)
│   (INCHANGÉ)            │
└──────────┬──────────────┘
           │ spawn
           ▼
┌─────────────────────────┐
│   gsd-executor          │  ← lit automatiquement .claude/rules/*
│   + execution-quality   │
│     rules auto-injectées│
└─────────────────────────┘
           │
           ▼
┌─────────────────────────┐
│   Fichiers modifiés     │
│   avec quality checks   │
│   appliqués             │
└─────────────────────────┘
```

**Principe clé :** Le gsd-executor lit déjà `./CLAUDE.md` et `.claude/rules/` à chaque spawn (confirmé ligne 132-133 de execute-phase.md). Aucune modification des fichiers GSD n'est nécessaire.

### Implementation Phases

#### Phase 1: Création du fichier rules (core)

**Livrable :** `.claude/rules/execution-quality.md`

**Contenu prévu (3 sections) :**

**Section 1 — System-Wide Test Check**
- Clause d'applicabilité : "S'applique quand la task modifie du code applicatif ou infrastructure. Skip pour docs-only/config-only."
- Table des 5 questions (extraite de ce-work) :
  - What fires when this runs? (callbacks, middleware, observers)
  - Do my tests exercise the real chain? (mocks vs integration)
  - Can failure leave orphaned state? (idempotency, cleanup)
  - What other interfaces expose this? (parity check)
  - Do error strategies align across layers? (retry conflicts)
- Skip heuristic : "Leaf-node changes with no callbacks, no state persistence, no parallel interfaces → skip in 10 seconds"

**Section 2 — Post-Deploy Monitoring**
- Directive conditionnelle : "Pour tout changement touchant du code en production, documenter dans le commit message ou SUMMARY.md"
- Checklist minimale : logs/search terms, metrics/dashboards, expected healthy signals, failure signals + rollback trigger
- Escape hatch : "Si aucun impact runtime : 'No monitoring needed: [raison en 1 ligne]'"

**Section 3 — Commit Quality Heuristics**
- Clause de précédence : "Ces heuristiques complètent le protocole de commit GSD. En cas de conflit, le protocole GSD prévaut."
- Table quand commit vs attendre (extraite de ce-work) :
  - Commit : logical unit complete, tests pass, context switch imminent, about to attempt risky changes
  - Wait : partial unit, tests failing, purely scaffolding, would need "WIP" message
- Heuristique : "Can I write a commit message that describes a complete, valuable change? If yes, commit."

**Success criteria :**

- [x] Fichier < 60 lignes (42 lignes)
- [x] Clause d'applicabilité en tête (scope gate)
- [x] Clause de précédence GSD explicite
- [x] 3 sections avec tables/checklists concises

#### Phase 2: Validation du chargement

**Livrable :** Preuve que le fichier est bien injecté dans les subagents

**Tasks :**
- [x] Vérifier que `.claude/rules/execution-quality.md` apparaît dans les system-reminders d'un subagent (via un test de spawn)
- [x] Confirmer qu'il n'y a pas de conflit avec `tool-routing.md` ou `flywheel-workflow.md`
- [x] Vérifier que le fichier ne dépasse pas le budget token raisonnable (~800-1000 tokens) — 630 tokens

**Success criteria :**
- [x] Subagent confirme avoir lu les rules
- [x] Pas de conflit avec les rules existantes
- [x] Token cost < 1000 tokens (630 tokens)

#### Phase 3: Documentation et intégration flywheel

**Livrable :** Mise à jour de la documentation projet

**Tasks :**
- [x] Ajouter une décision dans MEMORY.md : "Quality checks ce-work injectés via rules, pas en remplacement de GSD"
- [x] Ajouter dans CLAUDE.md.template une mention de execution-quality.md dans la section closure protocol
- [ ] Optionnel : ajouter une entrée LESSONS.md si le pattern est validé comme cross-projet

**Success criteria :**
- [x] MEMORY.md décision documentée
- [x] CLAUDE.md.template référence le fichier

## Alternative Approaches Considered

| Approche | Avantages | Inconvénients | Verdict |
|----------|-----------|---------------|---------|
| **A. Rules file (choisi)** | Auto-injecté, pas de modification GSD, léger | S'applique à TOUS les subagents (pas seulement executor) | Retenu — scope gate mitige le bruit |
| **B. Modifier execute-plan.md** | Ciblé au gsd-executor uniquement | Écrasé au prochain `gsd:update`, maintenance lourde | Rejeté |
| **C. Créer un .claude/skills/execution-quality/** | Chargement conditionnel par trigger | Les subagents ne chargent pas les skills par trigger, seulement par instruction explicite | Rejeté |
| **D. Remplacer gsd:execute-phase par ce-work** | Quality checks natifs | Perte state tracking, waves, verification, auto-advance | Rejeté (analyse détaillée dans conversation source) |
| **E. Patch GSD via gsd:reapply-patches** | Persiste après update | Complexité de maintenance des patches, fragile | Rejeté — rules/ est plus simple |

## System-Wide Impact

### Interaction Graph

```
Session start → Claude Code auto-loads .claude/rules/*.md
  → execution-quality.md injected into system-reminders
  → gsd-executor reads rules at task start
  → Before each task commit: executor checks system-wide test table
  → At plan completion: executor includes monitoring plan in SUMMARY.md
```

Aucun callback, middleware, ou observer — c'est un fichier texte passif lu par le LLM.

### Error & Failure Propagation

- **Fichier mal formaté :** Claude Code ignore silencieusement les rules malformées → risque = rules pas appliquées, pas de crash
- **Conflit avec GSD :** La clause de précédence explicite résout : "GSD protocol wins"
- **Token overflow :** Si le fichier dépasse ~60 lignes, il contribue au context flooding → mitigé par le budget strict

### State Lifecycle Risks

Aucun — le fichier ne persiste aucun état. Il est lu-seulement à chaque session.

### API Surface Parity

Les rules s'appliquent uniformément à :
- gsd-executor (cible principale)
- Main session (utile aussi)
- Tous les autres subagents (scope gate filtre les non-pertinents)

### Integration Test Scenarios

1. **Spawn un gsd-executor sur un plan test** → vérifier qu'il mentionne le system-wide test check dans son raisonnement
2. **Exécuter un plan docs-only** → vérifier que l'executor skip les quality checks ("docs-only, skipping")
3. **Créer un conflit de commit protocol** → vérifier que GSD protocol prévaut (commit par task atomique)
4. **Mesurer le token cost** → confirmer < 1000 tokens d'overhead

## Acceptance Criteria

### Functional Requirements

- [x] `.claude/rules/execution-quality.md` existe avec les 3 sections
- [x] Clause d'applicabilité en tête : skip pour docs/config-only
- [x] Clause de précédence GSD : complément, pas override
- [x] System-wide test check : table 5 questions avec skip heuristic
- [x] Post-deploy monitoring : checklist minimale avec escape hatch
- [x] Commit heuristics : table commit vs wait avec heuristique résumée

### Non-Functional Requirements

- [x] Fichier < 60 lignes (42 lignes)
- [x] Token cost < 1000 tokens (630 tokens)
- [x] Compatible avec les conventions existantes (style directif, markdown tables)
- [x] Pas de modification des fichiers GSD core

### Quality Gates

- [x] Subagent test confirme l'injection
- [x] Pas de régression sur les rules existantes
- [x] MEMORY.md et CLAUDE.md.template mis à jour

## Success Metrics

| Métrique | Mesure | Cible |
|----------|--------|-------|
| Adoption | gsd-executor applique les checks | Observable dans le raisonnement des executors |
| Token overhead | Taille du fichier en tokens | < 1000 tokens |
| Faux positifs | Checks inutiles sur tasks docs/config | 0 (scope gate fonctionne) |
| Conflits GSD | Erreurs ou contradictions de commit | 0 (clause de précédence fonctionne) |

## Dependencies & Prerequisites

- **Aucune dépendance bloquante** — le mécanisme `.claude/rules/` existe déjà et fonctionne
- **Prerequis :** comprendre le contenu exact des 3 patterns de ce-work (lu dans cette session)

## Risk Analysis & Mitigation

| Risque | Probabilité | Impact | Mitigation |
|--------|-------------|--------|------------|
| Rules ignorées par le LLM (trop de context) | Faible | Moyen | Budget strict < 60 lignes, style directif |
| Bruit sur tasks non-code | Moyen | Faible | Scope gate en tête du fichier |
| Conflit commit protocol | Faible | Moyen | Clause de précédence explicite |
| Écrasement au `gsd:update` | Nul | N/A | Le fichier est dans `.claude/rules/` du projet, pas dans GSD |
| Over-engineering du fichier | Moyen | Moyen | Cap strict à 60 lignes, pas de logique conditionnelle complexe |

## Resource Requirements

- **Effort :** ~15 minutes d'implémentation (1 fichier + 2 mises à jour doc)
- **Compétence :** Connaissance des conventions `.claude/rules/` et des patterns ce-work/GSD
- **Infrastructure :** Aucune

## Future Considerations

- Si le pattern est validé (quality checks via rules), le même mécanisme pourrait injecter d'autres patterns CE (PR workflow, screenshot capture, reviewer agent triggers)
- Si le fichier grandit au-delà de 60 lignes, découper en `execution-quality-testing.md` et `execution-quality-monitoring.md`
- Proposer le pattern comme contribution upstream à GSD (enrichir le gsd-executor prompt nativement)

## Documentation Plan

- [ ] MEMORY.md : décision "quality checks CE injectés via rules"
- [ ] CLAUDE.md.template : référence dans la section closure protocol
- [ ] LESSONS.md : si validé comme pattern cross-projet

## Sources & References

### Internal References

- Skill ce-work : [ce-work.md](~/.claude/commands/compound-engineering/ce-work.md) — lignes 102-115 (system-wide test check), 118-145 (commit heuristics), 206-215 (post-deploy monitoring)
- Skill gsd:execute-phase : [execute-phase.md](~/.claude/commands/gsd/execute-phase.md) — lignes 111-143 (executor prompt, confirme lecture CLAUDE.md)
- GSD workflow : [execute-phase.md](~/.claude/get-shit-done/workflows/execute-phase.md) — ligne 132 (subagent reads CLAUDE.md)
- Rules existantes : [tool-routing.md](.claude/rules/tool-routing.md), [flywheel-workflow.md](.claude/rules/flywheel-workflow.md)
- CLAUDE.md.template : closure protocol lignes 93-113

### Related Work

- Conversation source : session 2026-03-16 — analyse comparative gsd:execute-phase vs ce-work, décision hybride
- LESSONS.md : "deepening multi-agent prévient les bugs structurels"
