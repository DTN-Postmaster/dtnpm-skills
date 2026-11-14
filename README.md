# DTN Postmaster — Operator & User Skills (`skills`)

Catalogue de compétences (`SKILL.md`) destiné aux opérateurs, radioamateurs, administrateurs système et agents IA configurant ou exploitant **DTN Postmaster (`dtnpm`)** et ses utilitaires CLI.

> Pour la création de nouvelles extensions (`dtnpm-app-*`, `dtnpm-mod-*`, `dtnpm-cla-*`) ou les audits de développement Rust/TypeScript, consultez le dépôt compagnon **`skills-dev`**.

## Skills inclus

| Skill | Dossier | Rôle |
| --- | --- | --- |
| `dtnpm-station-setup` | `skills/dtnpm-station-setup/SKILL.md` | Mise en route d'une station `dtnpm` (`init`, `generate-config`, profils Cargo, TOML/YAML, stockage & quotas, routage, WebUI) |
| `dtnpm-cla-radio-config` | `skills/dtnpm-cla-radio-config/SKILL.md` | Configuration des adaptateurs RF et réseau (`hqfbp` KISS, `mercury-arq`, `ltp` UDP/KISS, `tcpclv4`) et opt-ins explicites |
| `dtnpm-cli-utilities` | `skills/dtnpm-cli-utilities/SKILL.md` | Manuel des utilitaires CLI (`dtnsend`, `dtnprint`, `dtnping`, `dtnfiles`, `dtnquery`, `dtntrigger`, `dtnforward`, `dtnbib`, `hdy-stats`, `dtnbasket`, `dtnbasket-cli`) |
| `dtnpm-iam-bpsec-security` | `skills/dtnpm-iam-bpsec-security/SKILL.md` | Sécurité IAM & BPSec (`dtnpm user`, `dtnpm token`, `[grpc.auth]` UDS, `[auth]` proxy/local, `keystore.toml`, signature/vérification) |
| `dtnpm-systemd-deployment-diagnostics` | `skills/dtnpm-systemd-deployment-diagnostics/SKILL.md` | Déploiement systemd durci (`dtnpm.service`), permissions `0600`/`0700`, reverse proxy YunoHost/Nginx, diagnostic (`dtnquery`, `hdy-stats`, `journalctl`) |

## Installation & découverte

### Dans un espace Antigravity (`.agents/skills.json` ou `skills.json`)

Ce dépôt fournit `skills.json` et `.agents/skills.json` pointant sur `skills/`.

1. Référencer le chemin dans le `skills.json` de votre environnement d'exploitation :
   ```json
   {
     "entries": [
       { "path": "../skills/skills" }
     ]
   }
   ```
2. Ou copier / lier les répertoires souhaités dans `~/.gemini/antigravity/skills/`.

## Validation locale & CI

```bash
./scripts/install-hooks.sh
./scripts/validate-skills.sh
```

La CI GitHub (`.github/workflows/ci.yml`) utilise exclusivement des actions épinglées par SHA-1 complet et `step-security/harden-runner` avec `permissions: contents: read`.

## Licence

Voir [LICENSE.md](LICENSE.md).
