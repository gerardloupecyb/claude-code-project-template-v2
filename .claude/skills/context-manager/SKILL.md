---
name: context-manager
description: >
  Règles de gestion du contexte, de la mémoire et du chain of thought pour
  Claude Code. Charger automatiquement sur tout projet qui utilise MEMORY.md,
  Supermemory, ou qui implique des sessions longues. Se déclenche aussi quand
  l'utilisateur mentionne "contexte", "mémoire", "reprendre le projet",
  "session", ou "MEMORY.md".
---

# Context Manager

Règles pour maintenir la qualité des réponses sur des sessions longues,
gérer la mémoire entre sessions, et structurer le raisonnement sur des
tâches complexes.

---

## Règles critiques — s'appliquent toujours

### Démarrage de session

TOUJOURS lire avant de coder ou planifier :

1. memory/MEMORY.md          → état courant, décisions actives
2. LESSONS.md                → leçons apprises, à appliquer immédiatement

Supermemory et docs/solutions/ sont consultés uniquement lors de la planification,
pas à chaque début de session (économie de tokens).

Si MEMORY.md n'existe pas → le créer depuis le template avant de continuer.
Si LESSONS.md n'existe pas → le créer depuis le template avant de continuer.
Ne jamais supposer le contexte. Toujours le lire.

### Fin de session

Mettre à jour MEMORY.md avant de fermer :

- Ce qui a été fait (3-5 lignes max)
- Décisions prises + raison courte
- Une seule prochaine étape, claire et actionnable
- Blocages ou questions ouvertes

Si la session a résolu un problème non-trivial → proposer `/lesson` pour capturer.
Commiter MEMORY.md et LESSONS.md dans le même commit que le code produit.

---

## Gestion des sessions longues

### Détecter la dégradation du contexte

Signaux d'alerte :

- Claude répète des questions déjà répondues plus tôt dans la session
- Réponses moins précises ou qui ignorent des contraintes établies
- Contexte à ~60-70% de capacité utilisée
- Incohérences avec des décisions prises en début de session

Action quand signal détecté :
Annoncer : "Contexte à [X]% — checkpoint recommandé avant de continuer."

### Protocole checkpoint

1. Résumer l'état en moins de 200 mots dans MEMORY.md
2. Lister les décisions prises depuis le début de session
3. Identifier la prochaine tâche (une seule, actionnable)
4. Proposer d'ouvrir une nouvelle session avec MEMORY.md en contexte initial

Règle absolue : ne jamais continuer une session dégradée.
La qualité se dégrade exponentiellement passé un certain seuil.
Une coupure propre avec MEMORY.md à jour produit de meilleurs résultats.

---

## Chain of thought — externaliser avant d'exécuter

### Quand l'appliquer

Obligatoire pour :

- Décisions d'architecture ou de structure de données
- Logique métier complexe (règles conditionnelles multiples)
- Debug difficile (plus d'une hypothèse possible)
- Choix entre plusieurs approches techniques

Optionnel pour :

- Tâches CRUD simples et bien définies
- Code boilerplate standard
- Modifications mineures sans impact architectural

### Format chain of thought

```
Problème : [ce qu'on résout exactement, pas ce qu'on va faire]
Contraintes : [limites connues — perf, API, sécurité, budget, etc.]
Options :
  A) [option] → avantage : [X] / inconvénient : [Y]
  B) [option] → avantage : [X] / inconvénient : [Y]
Choix : [option] parce que [raison courte et précise]
Risques : [ce qui pourrait mal tourner et comment on le détecte]
```

Où mettre ce bloc :

- Dans le PLAN.md GSD si c'est une tâche planifiée
- En commentaire en tête du fichier si c'est de l'architecture
- Dans MEMORY.md sous "Décisions actives" si c'est un choix durable

---

## Supermemory — archive principale par projet

Supermemory sert d'archive des leçons, organisée par projet.
Les leçons y arrivent via `/lesson migrate` (quand LESSONS.md atteint le cap 50).
Ne pas sauvegarder directement dans Supermemory — passer par `/lesson` ou `/workflows:compound`.

### Tags standards

- `[lesson:{domaine}]`        → leçon projet migrée depuis LESSONS.md
- `[skill:{domaine}]`         → règle technique réutilisable
- `[decision:architecture]`   → choix structurant cross-projets
- `[decision:stack]`          → choix de librairie ou outil
- `[lesson:error]`            → erreur à ne pas répéter
- `[convention:workflow]`     → façon de travailler à retenir
- `[context:preference]`      → préférence personnelle de travail

---

## Hiérarchie des couches mémoire (cache L1→L4)

| Couche | Contenu | Accès |
|--------|---------|-------|
| CARL rules | Règles critiques one-liner | Auto-injecté chaque prompt |
| LESSONS.md | Leçons récentes (quand/faire/parce que, cap 50) | Lu chaque session |
| Supermemory (projet) | Leçons archivées + résumés structurés | `recall` à la planification |
| docs/solutions/ | Patterns complets + code (backup git) | Agent search (fallback ou profondeur) |

| Information | Où la mettre |
|-------------|-------------|
| État courant du projet, où on en est | memory/MEMORY.md |
| Décisions actives qui influencent le code | memory/MEMORY.md → Décisions actives |
| Leçon apprise récente | LESSONS.md (via `/lesson`) |
| Leçon critique ou répétée | CARL rule (promotion via `/lesson`) |
| Leçon archivée (cap LESSONS.md atteint) | Supermemory projet + docs/solutions/ |
| Pattern détaillé avec code | docs/solutions/ (via `/workflows:compound`) |
| Code et implémentation | Git |
| Credentials et secrets | .env (jamais dans mémoire) |

Ne jamais mélanger les couches. Une leçon dans MEMORY.md doit migrer
vers LESSONS.md via `/lesson`.

---

## Anti-patterns

- Commencer à coder sans lire MEMORY.md et LESSONS.md
- Continuer une session dont le contexte est dégradé
- Sauvegarder directement dans Supermemory (passer par `/lesson` ou `/workflows:compound`)
- Dupliquer une leçon dans LESSONS.md au lieu de mettre à jour l'existante
- Stocker credentials ou données clients dans MEMORY.md, LESSONS.md, docs/, ou Supermemory
- Faire du chain of thought en tête implicitement sans l'externaliser
- Créer un domaine CARL avec moins de 3 règles distinctes
- Ignorer LESSONS.md lors de la planification

---

## Références

- memory/MEMORY.md                    → état courant projet
- LESSONS.md                          → cache chaud des leçons (cap 50)
- Supermemory (projet)                → archive principale des leçons
- docs/solutions/                     → backup local + patterns détaillés
- .carl/{domaine}                     → règles critiques injectées
- CLAUDE.md du projet                 → règles flywheel complètes
