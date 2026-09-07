---
name: oniria-package-dev
description: Règles de développement de packages ONIRIA V3. Utiliser quand on crée/modifie un package (oniria.package.json, oniria.build.ts, intentions, ressources, vues), on build/valide/pack un package (oniria package ..., build-package.mjs). Le déploiement CI/CD est décrit par le champ deploy du repo (mécanisme du repo), pas ici.
---

# Développement de packages ONIRIA

Références normatives (dans le repo oniria, espace Coder ONIRIA) :
- `docs/regles-developpement-oniria.md` — règles de dev (norme unique)
- `docs/guide-manifeste-package-oniria.md` — manifeste V3
- Référentiel ONIRTECH `01/02/03-architecture-code-coding-standards-enforcement` (hexagonal + DDD, pour le code TypeScript du package)
- Scripts : `scripts/oniria-package.ts`, `scripts/oniria-package-tooling.ts`, `scripts/build-package.mjs`

> Emplacement du code : `/home/coder/oniria` cité par un utilisateur = chemin dans le workspace Coder ; le traduire via le MCP `coder-workspaces` (`workspace_resolve`) avant d'agir.

> Le déploiement du projet et des packages est décrit par le **mécanisme CI/CD du repo**
> (champ `deploy` du repo `oniria`, fourni en contexte à l'orchestrateur ; workflows
> GitHub Actions `core-build-deploy.yml` / `package-build-deploy.yml`). Ce skill ne
> porte que les règles de DÉVELOPPEMENT.

---

## Règles de développement de package

### Principes invariants
- **Package V3** = unique unité publique d'extensibilité et de cycle de vie. Jamais d'« extension » comme notion publique.
- Le **cœur ne connaît aucun domaine de package** : aucune route, table, registre, libellé ou import spécifique à `pbn-network`, `chatbot-management`, `infomaniak-provider`, etc.
- Le **manifeste** (`oniria.package.json`) déclare TOUTES les contributions publiques. Si ce n'est pas dans le manifeste actif, ça n'existe pas.
- Une **séparation interne** = module privé du package (pas de manifeste, ni installation, ni permissions). Une capacité qui évolue/installe indépendamment devient un package dépendance.
- Cible obligatoire : **ONIRIA V3**, sans chemin de compatibilité avec l'ancien modèle.

### Structure d'un package
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

### Manifeste V3 — points clés
- `id` stable, unique, kebab-case (`^[a-z][a-z0-9-]*$`) ; `version` semver.
- `compatibility` : core, sdk, node, packageHostProtocol (`oniria.package-host.v1`).
- Contributions : `resources`, `intentions`, `views`, `navigation`, `events`, `httpEndpoints`, `objectStorage`, `serviceDependencies`, `dependencies`.
- `ui` : mode `web-component`, `entry`, `element` (`oniria-<id>`), `integrity` (sha256 auto-synchronisé au build).
- `intentions` : `name`, `entrypoint` (`./dist/server/index.js#exportName`), schémas entrée/sortie, `requiredScopes`, `executionModes`. **Interdit** : champ `scheduling`.
- Tous les chemins relatifs au package ; chemins absolus, `..`, séparateurs Windows interdits.
- Une **version publiée est immutable** : toute modification incrémente la version.

### Cycle de développement (chaîne officielle)
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

### Le Client ONIRIA (seule façade)
- Créé dans un entrypoint serveur : `createOniriaClient()` — **sans** passer le package, l'utilisateur ou l'organisation (le Package Host les injecte).
- CRUD métier → `oniria.resources` ; projection/requête → `resources.queries` ; mutations atomiques → `resources.transactions` ; règle métier/effet externe → intention V3 ; traitement long → `oniria.jobs` ; flux binaire/SSE/webhook → endpoint V3 ; secrets → `oniria.secrets`.
- **Interdit** de contourner le client (Supabase direct, repository du cœur, route métier interne).
- Chaque port/dépendance utilisé doit être déclaré dans le manifeste.
- Permissions **P22** appliquées côté serveur avant toute opération ; masquer un bouton n'est pas une protection.

### Tests & Definition of Done
- Tests attendus : unité (règles métier pures), intégration (client, permissions, migrations), Playwright (vues/erreurs), garde d'architecture (imports/routes/registres interdits).
- DoD : manifeste V3 valide + version incrémentée ; bundles autonomes ; aucun import/route/registre dans le cœur ; erreurs+idempotence+audit+secrets traités ; migrations/rollback vérifiés ; scénarios BDD verts ; `npm run oniria:build:check` synchronisé.
- Anti-patterns interdits : importer `@pbn/db` ou un repository métier depuis un package ; `SiteRepository` global ; CRUD dupliqué en intention ; secrets dans le manifeste ; publier sans incrément ; modifier le cœur pour enregistrer une vue/intention.

### Version d'un package — incrémenter AVANT de pousser (obligatoire)

Une **version publiée est immuable** et le CI applique un **anti-doublon** : déployer
une version déjà présente (stagée `var/oniria/packages/<id>/<version>/` ou installée
en base `p7_installed_manifests`) est **refusé**.

Quand tu modifies un package (fix/feature, ou toute modification de `oniria.package.json`,
`src/`, `migrations/`, `ui/`…) :

1. **Avant de créer/pousser ta branche `packages/<nom>`** : lis la version **courante
   sur la branche stable** `origin/oniria-preprod` (pas seulement ta branche locale) :
   `git fetch origin oniria-preprod`, puis
   `git show origin/oniria-preprod:packages/p7-ecosystem/src/extensions/<nom>/oniria.package.json | grep '"version"'`.
2. **Incrémente `version`** dans `oniria.package.json` : **patch par défaut**
   (`x.y.z → x.y.z+1`) ; mineur/majeur si la modification l'exige (breaking, feature
   majeure). Ne choisis **jamais** une version déjà publiée.
3. Vérifie que la nouvelle version est **strictement supérieure** à la dernière version
   disponible/active du package (dossiers stagés ou base) pour éviter le refus
   anti-doublon du CI.
4. Commit (modif + bump) puis push sur `packages/<nom>`. Si le CI refuse
   (anti-doublon) : soit c'est un re-run après échec partiel → `DEPLOY_FORCE_VERSION=1`,
   soit la version est déjà publiée → incrémente à nouveau.
5. Le déploiement (`deploy-package.mjs`) fait ensuite stage + **activation auto**
   (`DEPLOY_PACKAGE_OPERATOR_EMAIL`) — tu n'as pas à activer manuellement via le frontend.
