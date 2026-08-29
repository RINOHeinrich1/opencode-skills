---
name: task-execution
description: >-
  Use when orchestrating task execution across agents via the task-orchestrator
  MCP: registering tasks, driving the state machine (task_transition),
  reserving/releasing worktrees (lease + lifecycle), delegating to planner /
  executor agents (auditor agents only on explicit user request), following
  progress and validating. Trigger on words
  like "orchestrer", "orchestration", "coordonner les agents", "lancer la
  tâche", "task registry", "état d'avancement des tâches", "réserver un
  worktree", "machine à états", "quel travail est en cours".
---

> **Norme de référence** : `docs/norme-environnement-travail.md` (v1.0).
> Ce skill s'applique dans ce cadre : tout travail de développement passe par le
> workspace Coder du projet, jamais l'hôte.

# Skill: task-execution

Ce skill décrit la **méthode et les normes** d'orchestration de tâches sur les
projets, portées par l'agent `orchestrator` et le MCP `task-orchestrator`.

## Séparation des rôles

| Niveau | Rôle |
|---|---|
| **Agent** (`orchestrator`) | Responsabilité + boucle de raisonnement (déléguer, décider). |
| **Skill** (ce fichier) | Connaissance, méthode, procédure. |
| **MCP** (`task-orchestrator`, `coder-workspaces`, `plan-manager`, `audit-manager`) | Capacités techniques. |

## Machine à états (source de vérité : `task-orchestrator`)

```
queued → planning → awaiting_validation → (approved | rejected) → planned
       → in_progress → validating → review → (approved | rejected)
       → merge_pending → merged → deploy_pending → deploying
       → deployed → post_deploy_verified → done

latéraux : blocked · failed · aborted · crashed · rework
```

- Seul `task_transition` change un statut ; toute transition non listée est refusée.
- `decision_resolve` (kind=validation/review) provoque la **transition atomique**
  vers `approved`/`rejected` (statuts réservés à l'humain) + événement `CLOSED`.
- Les agents de fond publient des **événements** (`task_event`), jamais des états.

## Cycle de vie des worktrees

```
AVAILABLE → RESERVED → IN_USE → RELEASED → AVAILABLE
```

- `worktree_reserve` pose un **lease** (`leaseUntil`), renouvelé par `lease_renew`.
- `lease_expired` liste les worktrees récupérables (lease dépassé).
- Le verrou **physique** (git) reste géré par `session-guard.mjs` ; le registre
  porte l'état **logique** (qui possède quoi, pour quelle tâche, depuis quand).

## Sources de vérité

| Élément | Source de vérité |
|---|---|
| Tâches / réservations | MCP `task-orchestrator` (registre SQLite) |
| Plans | `plan-manager` (plans immuables + progression) |
| Audits | `audit-manager` |
| Code | Git / GitHub |
| Déploiement | pipeline CI/CD |

## Workflow standard

1. `task_register` (contexte + scope + dependencies) → `taskId`/`executionId`.
2. Résoudre le workspace Coder (`coder-workspaces`) ; jamais le code de l'hôte.
3. Détecter les conflits : `scope_conflict(project, scope)` (déterministe) + `worktree_list`.
4. Réutiliser ou créer un worktree (`session-guard`), puis `worktree_reserve`.
5. Déléguer : feature → `atomic-plan` (un plan par groupe d'objectifs
   interdépendants ; plusieurs plans = parallélisables, chacun enregistré via
   `plan_register`) puis exécution (`build-notify`) — **1 sous-tâche = 1 plan =
   1 exécution Build-Notify**, sous-tâches parallèles. Par sous-tâche terminée :
   commit → review/merge humain → déploiement (tant qu'il y a des fichiers
   ajoutés/modifiés/supprimés), sans attendre les autres sous-tâches.
   Audit d'architecture → `hexagonal-architecture-auditor`, **uniquement sur
   demande explicite de l'utilisateur** (tâche `type="audit"` à `task_register`).
   **Jamais d'audit automatique** sur une tâche feature/debug.
5bis. **Incohérence code↔plan** relevée par `build-notify`
   (`INCONSISTENCY_FOUND`) → consigner l'incohérence (`inconsistency_create` sur
   l'ancien plan, immuable) puis demander la rectification : nouveau plan
   `atomic-plan` + re-validation.
6. Suivre via `task_get`/`events_list` ; renouveler le lease.
7. **Par sous-tâche** : valider (`validating`) ; `decision_request(kind="review",
   by=<agent>, sessionId=<session>)` → décision humaine → `decision_resolve`
   (approved/rejected) → l'orchestrateur reprend (`approved` → merge ; `rejected`
   → rework).
8. **Par sous-tâche mergée** : déployer via CI/CD — `deployment_record` +
   transitions `deploy_pending → deploying → deployed → post_deploy_verified`
   (jamais de déploiement manuel).
9. `worktree_release` + rapport + email.

## Outils complémentaires (déploiement / scope / décisions)

| Outil | Usage |
|---|---|
| `deployment_record` | Tracer le déploiement CI/CD (statut, pipelineUrl, vérification). |
| `scope_conflict` | Détecter les chevauchements de périmètre entre tâches actives/worktrees. |
| `decision_request` / `decision_resolve` / `decision_expired` | Cycle de vie d'une décision humaine (échéance, escalade). |
| `participant_add` | Enregistrer un agent comme participant d'une tâche (idempotent). |

## Normes

- **Statuts de tâche** : voir machine à états ci-dessus.
- **Événements types** : `PLAN_CREATED`, `EXECUTION_STARTED`, `CHECKPOINT`,
  `INCONSISTENCY_FOUND`, `BLOCKED`, `EXECUTION_COMPLETED`, `VALIDATION_FAILED`,
  `AUDIT_COMPLETED`.
- **Priorités** : `low · normal · high · critical`.
- **Types de tâche** : `feature · debug · audit`.
- **Sous-tâche** : `1 sous-tâche = 1 plan = 1 exécution Build-Notify` ; les
  sous-tâches sont parallèles et chacune suit son cycle complet (commit →
  review/merge humain → déploiement) sans attendre les autres.
- **Participants** : tout agent utilisé sur une tâche est enregistré via
  `participant_add(taskId, agent, role)`.
- **Décisions** : toute décision inclut le détail (question/permission/éléments),
  l'agent demandeur (`by`), la session (`sessionId`) et la résolution
  (`decision_resolve`). Pour validation/review, la résolution est **couplée** à la
  transition vers `approved`/`rejected` + événement `CLOSED` (remarques).
- Ne mets jamais de secrets dans le registre, les documents ou les emails.
