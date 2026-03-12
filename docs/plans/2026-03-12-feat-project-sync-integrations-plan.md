---
title: "feat: Add project-sync skill + integrations.md"
type: feat
date: 2026-03-12
---

# project-sync — Synchronisation de l'état projet avec les outils externes

## Overview

Le template maintient l'état du projet dans MEMORY.md, GSD STATE.md, et docs/plans/. Mais **aucun mécanisme ne synchronise cet état avec les outils externes** (Linear, etc.) ni ne valide que les plans/roadmap sont cohérents avec ce qui a été fait.

Deux problèmes distincts à résoudre :

1. **Déclaration des intégrations** — chaque skill détecte les outils disponibles de façon ad hoc. Un seul fichier doit faire autorité sur ce qui est actif dans le projet.
2. **Synchronisation projet** — en fin de session, le travail accompli doit remonter dans les outils de gestion de projet (Linear) et les plans/roadmap doivent rester cohérents.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│ .claude/integrations.md                             │
│  Déclare : linear: true/false, gsd: true/false, …   │
│  Source de vérité unique pour tous les skills        │
└────────────────┬────────────────────────────────────┘
                 │ lu par
                 ▼
┌─────────────────────────────────────────────────────┐
│ project-sync (skill)                                │
│  Lit MEMORY.md + integrations.md                    │
│  Rapport advisory → actions confirmées par user     │
│  Advisory, jamais bloquant. N'exécute rien sans      │
│  confirmation explicite.                             │
└─────────────────────────────────────────────────────┘
```

### Design Principles

**Advisory avec confirmation.** project-sync propose des actions (créer une issue, marquer done), l'utilisateur confirme avant exécution. Aucune écriture sans confirmation explicite.

**integrations.md = source de vérité.** Tous les skills futurs lisent ce fichier. Fini la détection par tentative d'appel MCP.

**Découplé de session-gate.** session-gate valide la structure MEMORY.md. project-sync synchronise l'état vers l'extérieur. Responsabilités distinctes.

---

## Composant 1 — integrations.md

### Emplacement

`.claude/integrations.md` — dans `.claude/` pour rester dans l'espace du template, pas du projet métier.

### Format

```markdown
# Intégrations actives

> Fichier de configuration des outils externes. Lu par tous les skills.
> Généré par init-project.sh — modifier selon les outils actifs sur ce projet.

linear: {{LINEAR_ACTIVE}}
linear_team_id: {{LINEAR_TEAM_ID}}
linear_project_id: {{LINEAR_PROJECT_ID}}

gsd: {{GSD_ACTIVE}}

supermemory: {{SUPERMEMORY_ACTIVE}}
```

Format `clé: valeur` par ligne — greppable par tous les skills sans parser de YAML.

### Valeurs

- `true` / `false` pour les flags actifs
- IDs spécifiques pour les intégrations nécessitant une configuration (Linear team/project)
- `{{PLACEHOLDER}}` dans le template, rempli au setup ou laissé `false`

---

## Composant 2 — project-sync skill

### Les 4 vérifications

| # | Check | Condition | Mode | Sévérité |
|---|-------|-----------|------|----------|
| 1 | integrations.md lisible et sans placeholders | Toujours | START, END | [!!] |
| 2 | Linear : tâches "Ce qui a été fait" → issues à fermer | `linear: true` | END | advisory |
| 3 | Linear : "Prochaine étape" → issue existante ou à créer | `linear: true` | END | advisory |
| 4 | GSD : cohérence STATE.md ↔ MEMORY.md "Prochaine étape" | `gsd: true` | END | [--] |

### Check 1 — integrations.md valide (START, END)

Lire `.claude/integrations.md`. Vérifier :
- Fichier existe
- Aucune ligne contenant `{{` (placeholder non rempli)

Si absent : `[!!] integrations.md manquant — copier depuis template`
Si placeholders : `[!!] integrations.md contient des placeholders non remplis : [liste]`

### Check 2 — Linear : tâches à fermer (END, si `linear: true`)

Lire MEMORY.md section "Ce qui a été fait" (headings `###`).
Lire `linear_team_id` et `linear_project_id` depuis integrations.md.
Appeler `mcp__linear__list_issues` pour les issues ouvertes du projet.

Comparer : pour chaque heading `###` dans "Ce qui a été fait", chercher une issue Linear avec titre similaire.

Rapport :
```
[--] Linear — issues potentiellement à fermer :
     - "feat: session-gate skill" → issue #LAI-42 "Add session gate validation" [open]
     Confirmer pour marquer done ? (oui/non)
```

Si confirmation : appeler `mcp__linear__save_issue` pour marquer done + ajouter commentaire.

### Check 3 — Linear : prochaine étape couverte (END, si `linear: true`)

Lire MEMORY.md "Prochaine étape".
Chercher une issue Linear ouverte correspondante.

Si trouvée : `[ok] "Prochaine étape" couverte par issue #XX`
Si non trouvée :
```
[--] Aucune issue Linear pour : "Implémenter project-sync skill"
     Créer une issue ? (oui/non)
```

Si confirmation : `mcp__linear__save_issue` avec titre = prochaine étape.

### Check 4 — GSD cohérence (END, si `gsd: true`)

Lire STATE.md (phase courante, milestone).
Lire MEMORY.md "Prochaine étape".

Afficher : `[--] GSD Phase X.Y | Prochaine étape : [texte]` — informatif, pas de [!!].

---

## Format de sortie

```
Project Sync — END

  [ok]  integrations.md valide (linear: true, gsd: true)

  Linear :
  [--]  Issue à fermer : #LAI-42 "Add session gate validation" — "feat: session-gate" fait
        → Confirmer fermeture ? (oui)
  [ok]  Issue fermée : #LAI-42

  [--]  Prochaine étape sans issue : "Implémenter project-sync skill"
        → Créer issue ? (non)

  GSD :
  [--]  Phase 3.2 active | Prochaine étape : "Implémenter project-sync skill"

  Sync terminé.
```

Légende : `[ok]` = action confirmée, `[!!]` = erreur bloquante, `[--]` = informatif/question.

---

## Modes d'invocation

| Commande | Mode | Checks exécutés |
|---|---|---|
| `/project-sync` | END | 1, 2, 3, 4 (selon intégrations actives) |
| `/project-sync start` | START | 1 seulement |
| `/project-sync linear` | END | 1, 2, 3 |
| `/project-sync gsd` | END | 1, 4 |

---

## Trigger du skill

```yaml
---
name: project-sync
description: >
  Synchronise l'état projet avec les outils externes (Linear, GSD).
  Se déclenche sur : "project sync", "sync project", "synchroniser",
  "mettre à jour Linear", "sync linear".
  Peut aussi être invoqué explicitement avec /project-sync.
---
```

---

## Intégration dans le template

### Fichiers à créer

1. **`.claude/skills/project-sync/SKILL.md`** (~70-80 lignes)
2. **`.claude/integrations.md.template`** — template avec placeholders

### Fichiers à modifier

3. **`CLAUDE.md.template`** :
   - Table "Fichiers mémoire et état" : ajouter `.claude/integrations.md` comme fichier de config
   - Règle #2 (fin de session) : ajouter "Optionnellement, lancer `/project-sync` pour synchroniser avec les outils externes."

4. **`init-project.sh`** :
   - Copier `.claude/skills/project-sync/` dans le projet scaffoldé
   - Copier `.claude/integrations.md.template` → `.claude/integrations.md` avec sed substitution

### Fichiers NON modifiés

- `memory/MEMORY.md.template` (structure inchangée)
- `.claude/skills/session-gate/` (rôle distinct)
- `.carl/domain.template`

---

## Acceptance Criteria

```
AC-1: Given integrations.md avec linear: true et un linear_team_id valide,
      When `/project-sync` est invoqué après une session avec "Ce qui a été fait" rempli,
      Then le rapport liste les issues Linear candidates à fermer.

AC-2: Given une confirmation "oui" sur une issue à fermer,
      When project-sync exécute le sync,
      Then l'issue Linear est marquée done via mcp__linear__save_issue.

AC-3: Given integrations.md avec linear: false,
      When `/project-sync` est invoqué,
      Then les checks 2 et 3 sont skippés sans erreur.

AC-4: Given integrations.md avec des placeholders {{LINEAR_TEAM_ID}},
      When `/project-sync start` est invoqué,
      Then check 1 affiche [!!] avec la liste des placeholders non remplis.

AC-5: Given `init-project.sh` exécuté sur un nouveau projet,
      When le projet est créé,
      Then `.claude/integrations.md` existe avec les valeurs par défaut
      et `.claude/skills/project-sync/SKILL.md` existe.

AC-6: Given gsd: true et STATE.md présent,
      When `/project-sync` est invoqué,
      Then check 4 affiche [--] avec la phase GSD courante et la prochaine étape MEMORY.md.
```

---

## Boundaries

- `memory/MEMORY.md.template` (ne pas modifier)
- `.claude/skills/session-gate/` (ne pas modifier — rôle distinct)
- `.claude/skills/context-manager/` (ne pas modifier)
- `.claude/skills/pre-flight/` (ne pas modifier)
- Aucune écriture sans confirmation explicite de l'utilisateur

---

## Edge Cases

| Case | Handling |
|---|---|
| integrations.md absent | Check 1 = [!!] + message "copier depuis template" |
| `linear: true` mais linear_team_id = placeholder | Check 1 attrape le placeholder → [!!] |
| Aucune issue Linear dans le projet | Check 2 = `[ok] Aucune issue ouverte à fermer` |
| MCP Linear indisponible (non configuré) | Check 2/3 = `[--] MCP Linear non disponible — skip` |
| MEMORY.md "Ce qui a été fait" vide | Check 2 = skip (rien à fermer) |
| STATE.md absent avec `gsd: true` | Check 4 = `[--] STATE.md absent — GSD non initié` |
| Confirmation refusée | Aucune action exécutée, rapport affiché sans modification |

---

## Ce que project-sync ne fait PAS

- N'écrit rien sans confirmation explicite de l'utilisateur
- Ne remplace pas Linear comme outil de gestion de projet
- Ne lit pas la qualité des issues (pas de "cette issue est-elle bien écrite ?")
- Ne crée pas d'issues à partir de "Déviations d'exécution" (hors scope v1)
- Ne synchronise pas les docs/plans/ vers Linear (hors scope v1)
- Ne vérifie pas le ROADMAP.md GSD (couvert par `/gsd:progress`)
- N'est pas un remplacement de session-gate (rôles distincts)

---

## Estimation

| Composant | Taille | Complexité |
|---|---|---|
| integrations.md.template | ~15 lignes | Trivial |
| SKILL.md project-sync | ~70-80 lignes | Moyenne — MCP calls + confirmation flow |
| CLAUDE.md.template | ~5 lignes ajoutées | Trivial |
| init-project.sh | ~5 lignes | Trivial — copie + sed substitution |

**Total estimé :** ~100 lignes de contenu nouveau, 2 fichiers créés, 2 fichiers modifiés.
