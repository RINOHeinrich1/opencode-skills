---
name: git-linux-ops
description: >-
  Use for git history surgery/recovery and Linux system diagnostics: reflog,
  reset/revert/restore, rebase/cherry-pick, stash, worktrees, conflicts, or
  disk/process/port/service/network troubleshooting. Trigger on words like
  "git", "rebase", "cherry-pick", "reflog", "reset --hard", "conflit", "stash",
  "worktree", "df", "du", "systemctl", "journalctl", "port occupé", "permission
  denied", "chmod", "process". Compensates a known model weakness: verify state
  before any destructive command.
---

# Git & Linux — travailler juste (jamais à l'aveugle)

## Git : état AVANT toute action
Toujours commencer par : `git status` + `git diff` + `git log --oneline -10`.
Ne jamais lancer une commande destructive sans avoir lu l'état réel.

## Annuler / récupérer (du plus sûr au plus destructif)
- Restaurer un fichier : `git restore <fichier>` (ou `git checkout -- <fichier>`).
- Désindexer sans perdre les modifs : `git restore --staged <fichier>`.
- Défaire le dernier commit en gardant les modifs : `git reset --soft HEAD~1`.
- Défaire en déplaçant les modifs hors index : `git reset --mixed HEAD~1`.
- **DESTRUCTIF** (perd les modifs non commitées) : `git reset --hard` → à éviter sans confirmation.
- Annuler un commit déjà poussé : `git revert <sha>` (n'efface pas l'historique).
- Tout retrouver grâce au reflog : `git reflog` → `git reset --hard <sha>` / `git branch <nom> <sha>`.

## Réécriture d'historique
- Rebase interactif : `git rebase -i HEAD~n` (squash/fixup/reword/edit).
- Rejouer un commit : `git cherry-pick <sha>` (conflit → résoudre, `--continue` / `--abort`).
- Conflit : `git status` liste les fichiers ; `git diff` les zones ; puis `git add` + `--continue`.
- **Ne jamais force-push une branche partagée** ; si nécessaire sur une branche perso :
  `git push --force-with-lease` (jamais `--force` nu).

## Travail isolé
- `git stash` / `git stash pop` pour mettre de côté ; `git stash list`.
- Worktree : `git worktree add <chemin> <branche>` (un checkout par tâche, sans polluer le courant).
- Brancher depuis l'à-jour : `git fetch && git rebase origin/<branche>`.

## Linux : diagnostic méthodique
- Processus : `ps aux | grep <nom>`, `top`/`htop`.
- Port occupé : `ss -tlnp | grep <port>` ou `lsof -i :<port>`.
- Disque : `df -h` (volumes pleins), `du -sh <dir>/*` (qui consomme).
- Mémoire : `free -h` ; OOM : `dmesg | grep -i oom`.
- Services : `systemctl status <svc>`, logs `journalctl -u <svc> -n 100 --no-pager`.
- Réseau : `curl -v <url>`, `ping`, `dig <host>`.
- Permissions : `id`, `ls -l`, `stat <f>` ; corriger avec `chmod`/`chown` (jamais en root un
  workspace de dev — les agents n'exécutent pas en root).

## Règles d'or
1. Une commande destructive = lecture de l'état **puis** confirmation.
2. Ne jamais `rm -rf` un chemin construit par une variable non vérifiée.
3. Ne jamais committer de secret (`.env`, clés) ; vérifier `.gitignore`.
4. Après action : relire `git status` / `git log` pour confirmer le résultat.
