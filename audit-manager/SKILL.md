---
name: audit-manager
description: >-
  Use when managing the treatment of hexagonal architecture audit reports
  (produced by the hexagonal-architecture-auditor agent): registering immutable
  audit reports, marking the patch state of each violated rule (todo/in_progress/
  done/blocked/skipped), attaching incidents and inconsistencies as linked
  documents (never editing the audit report itself), generating status reports,
  and answering "what corrections are left to do". Trigger on words like
  "correction", "correctif", "état de patch", "traitement de l'audit",
  "avancement de l'audit", "incident", "incohérence", "correctifs restants",
  "marquer la règle", "clôturer l'incident".
---

# Skill: audit-manager

Ce skill décrit les **normes et outils** pour gérer le traitement des rapports
d'audit d'architecture hexagonale (produits par l'agent
`hexagonal-architecture-auditor`), via le serveur MCP `audit-manager`.

## Principe fondamental

- Les rapports d'audit (`audits/audit-arch_*.md`) sont **IMMUABLES**. Tu ne
  modifies JAMAIS le rapport.
- Toute information de traitement est écrite dans des **documents rattachés**
  sous `audits/.audit-manager/` :
  - `progression/<auditId>.md` — état de patch par règle ;
  - `incidents/INC-###.md` — incidents de traitement ;
  - `inconsistencies/INCO-###.md` — incohérences détectées ;
  - `reports/report-<auditId>-<date>.md` — rapports de statut.
- La source de vérité interrogeable est `audits/.audit-manager/index.json`
  (gérée par le MCP, ne l'édite pas à la main).
- Les incidents/incohérences sont **reflétés** dans la table partagée
  `audit_notifications` du registre (PostgreSQL) pour que le daemon
  `opencode-notifier` puisse notifier l'utilisateur (v0.1.0).

## Localisation des rapports d'audit

- Dossier canonique : `<rootPath>/audits/` (créé par l'agent d'audit).
- Nommage : `audit-arch_<module|standard>_<composant>_<YYYYMMDD-HHMMSS>.md`.
- L'`auditId` = le nom de fichier sans l'extension `.md`.

## Outils MCP disponibles (serveur `audit-manager`)

| Tool | Usage |
|---|---|
| `audit_register` | Enregistrer un rapport d'audit (immuable), extraire les règles violées en traitements `todo`. |
| `audit_list` | Lister les audits + avancement/incidents/incohérences. |
| `audit_get` | Détail d'un audit (règles + progression + rattachés). |
| `treatment_update` | Marquer l'état de patch d'une règle (todo/in_progress/done/blocked/skipped). |
| `treatments_list` | **Liste des correctifs à effectuer** (règles non terminées). |
| `incident_create` | Créer un incident rattaché (doc + miroir `audit_notifications`). |
| `incident_resolve` | Clôturer un incident avec résolution (doc + miroir `audit_notifications`). |
| `incident_list` | Lister les incidents (filtrable par audit/statut). |
| `inconsistency_create` | Signaler une incohérence (doc + miroir `audit_notifications`). |
| `inconsistency_list` | Lister les incohérences (filtrable par audit). |
| `audit_report` | Générer un rapport de statut markdown. |

Tous les outils prennent `rootPath` (racine du projet contenant `audits/`).

## Workflow standard

1. **Enregistrer** un rapport fraîchement produit :
   `audit_register(rootPath, auditFile, objective?, treatments?)` — les ruleId
   sont auto-extraits du rapport si non fournis.
2. **Suivre les correctifs** : après chaque règle corrigée, `treatment_update`
   avec le bon statut. Ne marque `done` que si le correctif est réellement
   appliqué **et vérifié**.
3. **Incidents** : en cas de blocage/erreur pendant un correctif,
   `incident_create` (sévérité low/medium/high/critical, titre + description,
   `ruleId` si lié). À la résolution : `incident_resolve`.
4. **Incohérences** : en cas de contradiction détectée (règle vs règle, audit vs
   audit), `inconsistency_create`.
5. **Répondre à « quels correctifs reste-t-il ? »** : `treatments_list(rootPath,
   auditId?)` renvoie la liste des règles non terminées.
6. **Rapport** : `audit_report` à la fin (ou sur demande) pour un état consolidé.

## Normes

- **Statuts de correctif** : `todo` (à corriger) · `in_progress` (en cours) ·
  `done` (corrigé + vérifié) · `blocked` (bloqué → associer un incident) ·
  `skipped` (écart assumé, documenter pourquoi en note).
- **Sévérités d'incident** : `low` · `medium` · `high` · `critical`.
- **Incident vs incohérence** : un **incident** est un événement de traitement
  (erreur, blocage) ; une **incohérence** est une contradiction logique entre
  règles/audits ou un écart détecté avant/après correction.
- **IDs** : incidents `INC-###`, incohérences `INCO-###` (attribués par le MCP).
- Ne mets jamais de secrets/mots de passe dans les documents.

## Notifications (v0.1.0 — par la plateforme, jamais par le MCP)

- Les incidents/incohérences créent un **miroir** dans `audit_notifications`
  (registre PostgreSQL) : la notification email est dérivée par le daemon
  `opencode-notifier`. Le MCP `audit-manager` n'envoie **aucun email**.
- L'outil `notify` a été **supprimé** du serveur MCP. Ne cherche pas à envoyer
  d'email : écris les états (incident/incohérence/treatment) et le notifier
  fera le reste.

## Exemples de questions utilisateur → action

- « Quels correctifs reste-t-il ? » → `treatments_list`.
- « Où en est l'audit X ? » → `audit_get` (ou `audit_report`).
- « Marque ARCH-DOMAIN-PURITY comme corrigé » → `treatment_update(ruleId:"ARCH-DOMAIN-PURITY", status:"done")`.
- « On a un problème sur SMELL-FAT-CONTROLLER » → `incident_create`.
