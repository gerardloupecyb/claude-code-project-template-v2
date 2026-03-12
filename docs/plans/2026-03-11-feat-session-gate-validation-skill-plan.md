---
title: "feat: Add session-gate validation skill"
type: feat
date: 2026-03-11
deepened: 2026-03-11
reviewed: 2026-03-11
---

# Session-Gate — Validation d'état entre sessions

## Review Summary

**Reviewed on:** 2026-03-11
**Reviewers:** DHH-style, Kieran-style, Code-simplicity
**Changes applied:**

1. **Dropped YAML output block** — no consumer exists, Claude reads its own checklist
2. **Check 2 downgraded to informational** — displays age, no `[!!]` (avoids false positives on irregular projects)
3. **Check 3 detection method clarified** — explicit table-row skip logic, removed escalation at 5
4. **Parsing simplified** — case-insensitive grep instead of slug normalization, `###` counting only
5. **Check 7 logic made explicit** — "nothing staged = skip" in checks table
6. **Check 1 "non-empty" defined** — must contain at least one markdown heading
7. **Removed** — "checks supprimés" table, Research Insight boxes, source annotations, branch display edge case
8. **Estimation updated** — ~40-50 lines SKILL.md (down from 60-80)

---

## Overview

Les projets scaffoldés par project-template-v2 dépendent de MEMORY.md et STATE.md pour maintenir la continuité entre sessions. Aujourd'hui, **toute l'enforcement est au niveau prompt** — rien ne vérifie mécaniquement que les règles sont suivies. Si MEMORY.md n'est pas mis à jour, si des tâches sont silencieusement ajoutées/retirées, si des déviations ne sont pas loguées — personne ne s'en aperçoit.

**Résultat :** la qualité de la mémoire se dégrade silencieusement au fil des sessions. Le flywheel perd de la valeur.

## Architecture

```
┌──────────────────────────────────────────────┐
│ context-manager (instructions)               │
│  - Règle #1: lire MEMORY.md                  │
│  - Règle #2: mettre à jour MEMORY.md         │
│  - Règle #3: checkpoints                     │
└──────────────┬───────────────────────────────┘
               │ "as-tu fait ce que tu devais ?"
               ▼
┌──────────────────────────────────────────────┐
│ session-gate (validation)                    │
│  6 checks mécaniques + 1 info               │
│  Checklist [ok]/[!!] — advisory, jamais      │
│  bloquant. Aucun jugement sémantique.        │
└──────────────────────────────────────────────┘
```

Séparation claire : context-manager dit **quoi faire**, session-gate vérifie **si c'est fait**.

### Design Principle

**Mechanical checks only.** Tout check doit être vérifiable par lecture de fichier, grep, ou git status. Aucun check ne doit requérir de jugement sémantique sur la qualité ou la pertinence du contenu. Ceci prévient le scope creep et les faux positifs.

---

## Les 7 Checks

Tous mécaniquement vérifiables. Aucun ne requiert d'état cross-session ni de jugement sémantique.

| # | Check | Mode | Sévérité | Comment vérifier |
|---|-------|------|----------|------------------|
| 1 | MEMORY.md existe et non vide | START, END | [!!] | Fichier existe + contient au moins un heading markdown (`^#`) |
| 2 | Âge de "Dernière session" | START | info | Parser date YYYY-MM-DD, afficher l'âge ("3 days ago"). Pas de `[!!]`. |
| 3 | "Déviations d'exécution" vidée | START | [!!] | Dans la section, sauter les 2 premières lignes `\|` (header + separator), compter les lignes `\|` restantes. Count doit être 0. |
| 4 | "Ce qui a été fait" <= 5 entrées | START, END | [!!] | Compter les headings `###` dans la section. |
| 5 | "Prochaine étape" présente et non-placeholder | START, END | [!!] | Section existe + pas de `{{...}}` |
| 6 | "Dernière session" = aujourd'hui | END | [!!] | Date == YYYY-MM-DD du jour |
| 7 | MEMORY.md stagé avec le code | END | [!!] | `git status` : SI fichiers stagés ET MEMORY.md PAS parmi eux ALORS [!!]. SI rien n'est stagé ALORS skip (non applicable). |

---

## Parsing : règles de robustesse

### Merge conflicts (toujours en premier)

Avant tout parsing, scanner pour `^<{7}`, `^={7}`, `^>{7}`. Si trouvé : reporter "MEMORY.md contient des conflits de merge non résolus. Les résoudre avant de continuer." et sauter les checks restants.

### Matching des sections

Matcher les sections par **grep case-insensitive** sur les headers connus du template : "Dernière session", "Déviations d'exécution", "Ce qui a été fait", "Prochaine étape". Pas de normalisation d'accents ni de slugification — le template contrôle les headers.

### Comptage des entrées "Ce qui a été fait"

Une **entrée** = un heading `###` dans la section. C'est le format du template. Les bullets et sous-bullets ne comptent pas.

### Format de date

Le template MEMORY.md utilise `{{DATE}}`. `init-project.sh` le remplace par `YYYY-MM-DD`. Le skill accepte uniquement ce format. Si non parsable : `[!!]` avec message d'erreur explicite.

---

## Format de sortie

```
Session Gate — START

  [ok]  MEMORY.md exists and is non-empty
  [--]  "Dernière session" 2026-03-10 (1 day ago)
  [!!]  "Déviations d'exécution" has 2 entries — clear them
  [ok]  "Ce qui a été fait": 3/5
  [ok]  "Prochaine étape" present

  1 issue found. Fix before continuing.
```

Légende : `[ok]` = pass, `[!!]` = action requise, `[--]` = info (pas de sévérité).

S'il y a des `[!!]`, dire combien et recommander de fixer. Sinon, "All clear."

---

## Modes d'invocation

| Commande | Mode | Checks exécutés |
|---|---|---|
| `/session-gate start` | START | 1, 2, 3, 4, 5 |
| `/session-gate end` | END | 1, 4, 5, 6, 7 |
| `/session-gate` (sans arg) | BOTH | Tous les 7 |

---

## Trigger du skill

```yaml
---
name: session-gate
description: >
  Validation mécanique de l'état de session (MEMORY.md).
  Se déclenche sur : "session gate", "valider la session",
  "vérifier mémoire", "état de la session".
  Peut aussi être invoqué explicitement avec /session-gate.
---
```

---

## Intégration dans le template

### Fichiers à créer

1. **`.claude/skills/session-gate/SKILL.md`** (~40-50 lignes)

### Fichiers à modifier

2. **`CLAUDE.md.template`** :
   - Règle #1 : ajouter "Optionnellement, lancer `/session-gate start` pour valider l'état." (recommandé, pas obligatoire)
   - Règle #2 : ajouter "Optionnellement, lancer `/session-gate end` avant le commit."
   - Table "Fichiers partagés" : ajouter session-gate comme consommateur de MEMORY.md

3. **`init-project.sh`** :
   - Copier `.claude/skills/session-gate/` dans le projet scaffoldé

### Fichiers NON modifiés

- context-manager SKILL.md (instructions, pas validation)
- `.carl/domain.template` (RULE_4/5 restent complémentaires)
- `memory/MEMORY.md.template` (structure inchangée)

---

## Acceptance Criteria

```
AC-1: Given un projet scaffoldé avec MEMORY.md à jour,
      When `/session-gate start` est invoqué,
      Then le rapport affiche tous [ok]/[--] et "All clear."

AC-2: Given un MEMORY.md avec déviations non vidées et > 5 entrées,
      When `/session-gate start` est invoqué,
      Then le rapport affiche [!!] sur checks 3 et 4 avec actions.

AC-3: Given `init-project.sh` exécuté sur un nouveau projet,
      When le projet est créé,
      Then `.claude/skills/session-gate/SKILL.md` existe.

AC-4: Given un MEMORY.md avec date "Dernière session" il y a 12 jours,
      When `/session-gate start` est invoqué,
      Then check 2 affiche [--] "12 days ago" (info, pas [!!]).

AC-5a: Given des fichiers stagés mais MEMORY.md n'est PAS parmi eux,
       When `/session-gate end` est invoqué,
       Then check 7 affiche [!!] "Files are staged but MEMORY.md is not".

AC-5b: Given aucun fichier stagé (session brainstorm, pas de code),
       When `/session-gate end` est invoqué,
       Then check 7 est skippé (non applicable).
```

---

## Boundaries

- `.carl/domain.template` (ne pas modifier)
- `memory/MEMORY.md.template` (structure inchangée)
- `.claude/skills/pre-flight/` (ne pas modifier)
- `.claude/skills/context-manager/` (ne pas modifier)

---

## Edge Cases

| Case | Handling |
|---|---|
| MEMORY.md n'existe pas | Check 1 = [!!] + message "créer depuis template" |
| MEMORY.md a des merge conflicts | Pré-check : reporter et sauter les checks restants |
| Template MEMORY.md modifié (table Déviations restructurée) | Check 3 assume exactement 2 lignes `\|` avant les données (header + separator). **Invariant implicite** : si `memory/MEMORY.md.template` change la structure de cette table, Check 3 doit être mis à jour en conséquence. |
| Première session (placeholders) | Check 5 attrape `{{...}}` dans prochaine étape. Check 2 affiche l'âge sans flag. |
| Pas de GSD actif (pas de STATE.md) | Aucun impact — aucun check ne lit STATE.md |
| Session sans code (brainstorm only) | Check 7 : rien n'est stagé → skip |
| Date non parsable | [!!] avec message d'erreur explicite, pas de fallback silencieux |

---

## Estimation

| Composant | Taille | Complexité |
|---|---|---|
| SKILL.md | ~40-50 lignes | Faible — grep + date comparison + git status |
| CLAUDE.md.template | ~5 lignes ajoutées | Trivial |
| init-project.sh | ~3 lignes | Trivial — copie du dossier |

---

## Ce que session-gate ne fait PAS

- Ne modifie aucun fichier (read-only, comme pre-flight)
- Ne bloque jamais la session (advisory uniquement)
- Ne juge pas la qualité du contenu (pas de "la prochaine étape est-elle bonne ?")
- Ne lit pas STATE.md (évite le couplage fragile avec GSD)
- Ne requiert pas d'état entre invocations (stateless)
- Ne remplace pas les CARL rules 4/5 (les complète)
