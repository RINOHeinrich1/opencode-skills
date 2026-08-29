---
name: coder-workspace-locations
description: >-
  Use when the user references a path inside a Coder workspace (typically
  "/home/coder/...") or asks "où se trouve le projet", "où est le code", "dans
  coder", "workspace coder", "volume docker", "dossier du projet". Teaches the
  agent that /home/coder is NOT the host's /home/coder but a Docker volume
  mounted inside a Coder workspace container, and to translate the in-workspace
  path into the real host path via the coder-workspaces MCP tools.
---

# Localiser les projets développés dans Coder

Coder est l'IDE auto-hébergé : `https://ide.madatalk.fr` (serveur
`coder.service`, écoute `127.0.0.1:7080`, nginx en reverse proxy sur cette
machine).

## Le piège principal

Quand un utilisateur dit « le projet est dans `/home/coder/oniria` », **ce
n'est PAS `/home/coder` de la machine hôte**. C'est le `/home/coder`
**à l'intérieur d'un workspace Coder**, c'est-à-dire d'un conteneur Docker.
Sur l'hôte, ce dossier correspond à un **volume Docker nommé**
`coder-<workspace_id>-home`, monté à `/home/coder` dans le conteneur.

La machine hôte possède AUSSI son propre `/home/coder` (home de l'utilisateur
système `coder`, qui contient la config serveur Coder
`/home/coder/.config/coderv2`) : **ne jamais lire/écrire là pour le code du
projet**.

## Comment retrouver le projet (TOUJOURS dynamique)

Ne code **jamais** une liste de workspaces en dur : elle devient obsolète dès
qu'un workspace est recréé, supprimé, renommé ou migré. Utilise le MCP
`coder-workspaces`, qui interroge Docker à la volée.

### 1. Lister les workspaces disponibles

Appeler l'outil MCP `workspace_list` (serveur `coder-workspaces`). Il renvoie
pour chaque workspace : `name`, `owner`, `container`, `running`, `volume`,
`hostHome` (chemin hôte réel du `/home/coder`) et `projects` (dossiers
présents).

### 2. Traduire un chemin « /home/coder/… » en chemin hôte

Appeler `workspace_resolve` avec :

- `path` : le chemin tel que cité par l'utilisateur (ex. `/home/coder/oniria/package.json`).
- `workspace` (optionnel) : le nom/owner/conteneur pour lever l'ambiguïté.

Il renvoie `hostPath` (ex.
`/var/lib/docker/volumes/coder-…-home/_data/oniria/package.json`) et `exists`.

```text
/home/coder/<projet>/...   →   /var/lib/docker/volumes/coder-<workspace_id>-home/_data/<projet>/...
```

### 3. Plusieurs workspaces ? Demander confirmation

Si `workspace_list` renvoie plusieurs workspaces, ou si `workspace_resolve`
renvoie `ambiguous: true` avec une liste de `candidates`, **demander à
l'utilisateur** lequel il vise avant de toucher à un fichier. Ne jamais deviner.

## Accès direct (fallback sans MCP)

Si le MCP est indisponible, retrouver la correspondance manuellement :

```bash
docker ps -a --format '{{.Names}}' | grep '^coder-'          # conteneurs Coder
docker inspect coder-<owner>-<workspace> \
  --format '{{range .Mounts}}{{.Destination}} <- {{.Source}} ({{.Type}}){{println}}{{end}}'
docker volume ls --format '{{.Name}}' | grep '^coder-'       # volumes Coder
docker volume inspect <volume> --format '{{json .Labels}} mountpoint={{.Mountpoint}}'
```

Règles de nommage :

- Conteneur : `coder-<owner>-<workspace_name>` (ex. `coder-RINOHeinrich1-oniria`).
- Volume : `coder-<workspace_id>-home` (UUID lisible dans le label
  `coder.workspace_id` du volume).
- Mountpoint hôte : `/var/lib/docker/volumes/<volume>/_data`.

## Opérations courantes

```bash
# Lire un fichier du workspace depuis l'hôte (après workspace_resolve)
cat <hostPath>

# Exécuter une commande DANS le conteneur du workspace (vue identique au user)
docker exec -it <container> bash -lc 'ls /home/coder'
docker exec <container> bash -lc 'cd /home/coder/<projet> && git status'
```

## À retenir

1. `/home/coder/…` cité par un utilisateur ⇒ chemin **dans le conteneur du
   workspace**, à traduire via `workspace_resolve` (volume Docker `…/_data/…`)
   pour y accéder depuis l'hôte.
2. Le `/home/coder` de l'hôte (user `coder`) sert au serveur Coder, pas au code.
3. Ce dépôt (`/root/oniria-preprod`) est la **machine de préprod/déploiement** ;
   le code des développeurs vit dans les volumes des workspaces Coder.
4. Toujours découvrir via le MCP, jamais via une liste codée en dur ; en cas
   d'ambiguïté, demander confirmation.
