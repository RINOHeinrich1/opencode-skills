---
name: oniria-package-deploiement
description: Règles de développement de packages ONIRIA V3 et méthodes de déploiement du projet oniria. Utiliser quand on crée/modifie un package (oniria.package.json, oniria.build.ts, intentions, ressources, vues), on build/valide/pack un package (oniria package ..., build-package.mjs), ou on déploie le projet (pm2:deploy, pm2:preprod, install.sh, branche packages/<nom>, stage-package-assets, deploy-package, var/oniria/packages).
---

# Développement de packages ONIRIA & déploiement du projet

Références normatives (dans le repo oniria, espace Coder ONIRIA) :
- `docs/regles-developpement-oniria.md` — règles de dev (norme unique)
- `docs/regles-infrastructure-oniria.md` — exploitation/déploiement
- `docs/deploiement-packages-par-branches.md` — déploiement par package
- `docs/guide-manifeste-package-oniria.md` — manifeste V3
- Référentiel ONIRTECH `01/02/03-architecture-code-coding-standards-enforcement` (hexagonal + DDD, pour le code TypeScript du package)
- Scripts : `scripts/oniria-package.ts`, `scripts/oniria-package-tooling.ts`, `scripts/build-package.mjs`, `scripts/stage-package-assets.mjs`, `scripts/deploy-package.mjs`, `scripts/pm2-deploy.cjs`, `scripts/pm2-preprod.cjs`, `install.sh`

> Emplacement du code : `/home/coder/oniria` cité par un utilisateur = chemin dans le workspace Coder ; le traduire via le MCP `coder-workspaces` (`workspace_resolve`) avant d'agir. Le préprod hôte est `/root/oniria-preprod`.

---

## 1. Règles de développement de package

### 1.1 Principes invariants
- **Package V3** = unique unité publique d'extensibilité et de cycle de vie. Jamais d'« extension » comme notion publique.
- Le **cœur ne connaît aucun domaine de package** : aucune route, table, registre, libellé ou import spécifique à `pbn-network`, `chatbot-management`, `infomaniak-provider`, etc.
- Le **manifeste** (`oniria.package.json`) déclare TOUTES les contributions publiques. Si ce n'est pas dans le manifeste actif, ça n'existe pas.
- Une **séparation interne** = module privé du package (pas de manifeste, ni installation, ni permissions). Une capacité qui évolue/installe indépendamment devient un package dépendance.
- Cible obligatoire : **ONIRIA V3**, sans chemin de compatibilité avec l'ancien modèle.

### 1.2 Structure d'un package
```text
mon-package/
├── oniria.package.json        # manifeste V3 (obligatoire, racine, schemaVersion: 3)
├── oniria.build.ts            # config de build producteur (jamais exécutée à l'installation)
├── package.json
├── migrations/                # SQL ordonné, immutable ; appliquée → jamais réécrite
├── src/
│   ├── domain/
│   ├── application/
│   ├── server/                # intentions/, events/, endpoints/
│   ├── ui/                    # components/, views/, index.ts
│   └── shared/
├── dist/                      # bundles autonomes (server/ + ui/) — jamais versionnés à la main
└── tests/
```
Emplacement monorepo : `packages/p7-ecosystem/src/extensions/<name>/`.

### 1.3 Manifeste V3 — points clés
- `id` stable, unique, kebab-case (`^[a-z][a-z0-9-]*$`) ; `version` semver.
- `compatibility` : core, sdk, node, packageHostProtocol (`oniria.package-host.v1`).
- Contributions : `resources`, `intentions`, `views`, `navigation`, `events`, `httpEndpoints`, `objectStorage`, `serviceDependencies`, `dependencies`.
- `ui` : mode `web-component`, `entry`, `element` (`oniria-<id>`), `integrity` (sha256 auto-synchronisé au build).
- `intentions` : `name`, `entrypoint` (`./dist/server/index.js#exportName`), schémas entrée/sortie, `requiredScopes`, `executionModes`. **Interdit** : champ `scheduling`.
- Tous les chemins relatifs au package ; chemins absolus, `..`, séparateurs Windows interdits.
- Une **version publiée est immutable** : toute modification incrémente la version.

### 1.4 Cycle de développement (chaîne officielle)
Depuis la racine du monorepo (jamais de script public nommé d'après un domaine métier) :
```bash
npm run oniria:package -- create <id> [--template full|minimal|provider|ui] [--publisher <id>]
npm run oniria:package -- dev [<chemin>] [--once]       # watch + rebuild
npm run oniria:package -- build [<chemin>] [--check]    # bundle webpack node/web → dist/
npm run oniria:package -- validate [<chemin>]           # validation folder + release contract
npm run oniria:package -- test [<chemin>]
npm run oniria:package -- pack [<chemin>] [--output <dir>] [--skip-tests]  # .oniria.tgz + SHA-256 + .oniria.release.json
npm run oniria:package -- inspect <dossier|archive>
```
- `build --check` échoue si `dist/` est désynchronisé (CI). Build UI → calcule et synchronise `ui.integrity` dans le manifeste.
- `pack` : exige build + validate + tests, génère l'archive `release/<id>-<version>.oniria.tgz`.
- Validate : erreurs si entrypoint d'intention/endpoint n'exporte pas la fonction déclarée, bundle UI sans Custom Element, import `node:` dans l'UI, ou `dist/` contenant `.env`/`node_modules`/`src`.
- Les sorties de build doivent être sous `dist/` (sinon refusé).

### 1.5 Le Client ONIRIA (seule façade)
- Créé dans un entrypoint serveur : `createOniriaClient()` — **sans** passer le package, l'utilisateur ou l'organisation (le Package Host les injecte).
- CRUD métier → `oniria.resources` ; projection/requête → `resources.queries` ; mutations atomiques → `resources.transactions` ; règle métier/effet externe → intention V3 ; traitement long → `oniria.jobs` ; flux binaire/SSE/webhook → endpoint V3 ; secrets → `oniria.secrets`.
- **Interdit** de contourner le client (Supabase direct, repository du cœur, route métier interne).
- Chaque port/dépendance utilisé doit être déclaré dans le manifeste.
- Permissions **P22** appliquées côté serveur avant toute opération ; masquer un bouton n'est pas une protection.

### 1.6 Tests & Definition of Done
- Tests attendus : unité (règles métier pures), intégration (client, permissions, migrations), Playwright (vues/erreurs), garde d'architecture (imports/routes/registres interdits).
- DoD : manifeste V3 valide + version incrémentée ; bundles autonomes ; aucun import/route/registre dans le cœur ; erreurs+idempotence+audit+secrets traités ; migrations/rollback vérifiés ; scénarios BDD verts ; `npm run oniria:build:check` synchronisé.
- Anti-patterns interdits : importer `@pbn/db` ou un repository métier depuis un package ; `SiteRepository` global ; CRUD dupliqué en intention ; secrets dans le manifeste ; publier sans incrément ; modifier le cœur pour enregistrer une vue/intention.

---

## 2. Méthodes de déploiement du projet

### 2.1 Déploiement préproduction
```bash
export PM2_HOME=/root/.pm2
npm run pm2:preprod
```
Fixe port `3003`, app `oniria-preprod`, racine d'assets `var/oniria/packages` ; build + prépare artefacts V3 → hydrate → startOrReload idempotent → save PM2. **Un échec de build ne redémarre pas les processus existants.**

### 2.2 Déploiement standard
```bash
npm run build
npm run pm2:deploy
npm run pm2:status
```
`pm2:start`/`pm2:restart`/`pm2:deploy:all` passent par le même orchestrateur idempotent (`scripts/pm2-deploy.cjs`) : hydrate packages V3, warm avatar assets, supprime les processus inactifs, vérifie le contrat de lancement PM2 (`node_modules/next/dist/bin/next`, cwd, `ONIRIA_PACKAGE_ASSETS_ROOT`).

### 2.3 Processus supervisés (ecosystem.config.cjs)
| App | Rôle |
|---|---|
| `oniria-admin` (Next.js, port `ONIRIA_ADMIN_PORT`/`PORT`/3000) | HTTP, UI, API, dispatch |
| `oniria-runtime-scheduler` | détecte les échéances PBN Network et crée des jobs |
| `oniria-runtime-execution-runner` | revendique et exécute les jobs |

PM2 est un **superviseur, pas un contrat** : la santé d'un service = probe/heartbeat, pas la présence d'un PID.

### 2.4 Provisionnement initial (Ubuntu 24.04)
`./install.sh` : vérifie OS + Node ≥20.9 → apt deps → `.env` (modèle depuis `.env.example` sinon stop) → `npm ci --include=dev` → `extensions:ui` → `npm run build` → `pm2:deploy` → `pm2:save`. Options de skip : `SKIP_APT`, `SKIP_NPM_CI`, `SKIP_BUILD`, `SKIP_PM2_START`, `SKIP_PM2_SAVE`, `ALLOW_NON_UBUNTU`.

### 2.5 Déploiement d'un package par branche Git (sans toucher au projet)
Principe : **une branche = un package**.

```bash
git checkout packages/chatbot-management   # créer si absente : git checkout -b packages/chatbot-management origin/oniria-preprod
# ⚠️ incrémenter la version dans oniria.package.json AVANT de pousser (sinon écrasement du dossier de même version)
git add packages/p7-ecosystem/src/extensions/chatbot-management/
git commit -m "feat(chatbot-management): ..."
git push origin packages/chatbot-management
```
Workflow `.github/workflows/package-build-deploy.yml` :
1. Job `package-name` : extrait `<nom>` de `packages/<nom>`.
2. Job `test` (**gate**) sur ubuntu-latest : `npm ci` → `node scripts/build-package.mjs <package>` → `validate` (échec → déploiement bloqué).
3. Job `deploy` sur runner self-hosted `preprod-vm`, **push uniquement** : gate vert (polling check-runs, timeout 10 min) → `git pull` de la branche → build du seul package → **stage** `var/oniria/packages/<id>/<version>/` → `pm2 reload oniria-admin` (non bloquant).

**Dépôt ≠ installation** : le stage rend la version disponible sur le filesystem ; l'**activation** (version active en base) et la **mise à jour** du package (manifeste installé, release, migrations) se font **par l'administrateur via son panel** (frontend). Le pipeline ne fait pas de bump automatique et ne construit jamais les autres packages.
- Build par package : `scripts/build-package.mjs` (prérequis internes : `chatbot-management` régénère le bundle webpack `manager-handlers.generated.js` ; `pbn-network`/`infomaniak-provider` builds legacy ; sinon `oniria package build`).
- Stage manuel : `node scripts/run-ts.cjs scripts/stage-package-assets.mjs <package-dir>` (réutilise `stageV3PackageAssets` = hydratation d'un seul package, sans install DB).
- ⚠️ `deploy-package.mjs` fait `pm2 reload oniria-admin` ; sur l'hôte préprod l'app qui sert le trafic est **`oniria-preprod`** (port 3003). Ce reload cible la mauvaise app (no-op silencieux) — ne pas « réparer » manuellement hors périmètre agent.

### 2.6 Rôle de l'agent vs administrateur — activation & mise à jour du package (RÈGLE STRICTE)

**L'agent ne gère JAMAIS l'activation ni la mise à jour en base d'un package.** Ces opérations
sont réservées à **l'administrateur**, qui les effectue dans son **panel** (frontend d'administration) :
- **Activation de la version** : passage à la version `active` (`oniria_package_releases`,
  RPC `begin_oniria_package_release` → `transition_oniria_package_release` → `promote_oniria_package_release`) ;
- **Mise à jour du package** : inscription du manifeste dans le catalogue installé
  (`p7_installed_manifests`), `p7_installed_manifest_activations`, migrations applicatives.

En conséquence, l'agent :
- **NE fait PAS** d'INSERT/UPDATE/DELETE direct en base (`p7_installed_manifests`,
  `oniria_package_releases`, `oniria_package_release_transitions`, `p7_installed_manifest_activations`, …) ;
- **NE fait PAS** d'appels aux RPC de lifecycle (`begin_/transition_/promote_oniria_package_release`, …) ;
- **NE fait PAS** de `pm2 reload`/restart/start manuel des apps de l'hôte préprod ;
- **NE manipule PAS** le working tree du préprod (`/root/oniria-preprod`) hors du pipeline CI ;
- **N'interprète PAS** une version stagée non active comme un problème à corriger.

**Fin de déploiement côté agent** : après le push sur `packages/<nom>` et le passage du
workflow `package-build-deploy` (test + deploy = **stage**), l'agent **s'arrête** et informe
l'utilisateur que l'**activation / mise à jour de la version se fait par l'administrateur dans
son panel** (choix de la version active). Le dossier stagé `var/oniria/packages/<id>/<version>/`
est alors disponible pour cette opération.

> ⚠️ Le runtime ne sert que la version **active** (`oniria_package_releases.status='active'`,
> résolu via `isLifecycleSelectedVersion`). Tant que l'administrateur n'a pas activé la nouvelle
> version dans son panel, l'ancienne reste servie — c'est le comportement **attendu**, pas un bug.

### 2.7 Assets runtime
- Racine : `ONIRIA_PACKAGE_ASSETS_ROOT` → `<checkout>/var/oniria/packages` (identique entre build et runtime).
- Un dossier versionné contient : `dist/{server,ui}` + `oniria.package.json` + `package.json` sanitisé.
- Un bundle dont le hash ne correspond pas à la release est refusé : reconstruire + incrémenter, jamais modifier l'artefact hydraté.

### 2.8 Diagnostic (ordre)
1. `npm run pm2:status` + PID réel du port
2. cwd + environnement du processus
3. `ONIRIA_PACKAGE_ASSETS_ROOT`
4. existence du bundle source et hydraté
5. réponse HTTP asset/health check
6. version installée/active + migrations
7. logs corrélés (application, Package Host, scheduler, runner)
8. nginx seulement après validation de l'upstream

`502` = upstream absent/échec ; `404` d'asset V3 avec fichier présent = racine runtime/version active/processus obsolète incohérent.
