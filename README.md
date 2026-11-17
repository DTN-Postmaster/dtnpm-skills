# DTN Postmaster — Operator & User Skills (`skills`)

Catalog of skills (`SKILL.md`) intended for operators, amateur radio operators, system administrators, and AI agents configuring or operating **DTN Postmaster (`dtnpm`)** and its CLI utilities.

> To create new extensions (`dtnpm-app-*`, `dtnpm-mod-*`, `dtnpm-cla-*`) or run Rust/TypeScript development audits, see the companion repository **`skills-dev`**.

## Included Skills

| Skill | Directory | Role |
| --- | --- | --- |
| `dtnpm-station-setup` | `skills/dtnpm-station-setup/SKILL.md` | Setting up a `dtnpm` station (`init`, `generate-config`, Cargo profiles, TOML/YAML, storage & quotas, routing, WebUI) |
| `dtnpm-cla-radio-config` | `skills/dtnpm-cla-radio-config/SKILL.md` | Configuring RF and network adapters (`hqfbp` KISS, `mercury-arq`, `ltp` UDP/KISS, `tcpclv4`) and explicit opt-ins |
| `dtnpm-cli-utilities` | `skills/dtnpm-cli-utilities/SKILL.md` | CLI utilities manual (`dtnsend`, `dtnprint`, `dtnping`, `dtnfiles`, `dtnquery`, `dtntrigger`, `dtnforward`, `dtnbib`, `hdy-stats`, `dtnbasket`, `dtnbasket-cli`) |
| `dtnpm-iam-bpsec-security` | `skills/dtnpm-iam-bpsec-security/SKILL.md` | IAM & BPSec security (`dtnpm user`, `dtnpm token`, `[grpc.auth]` UDS, `[auth]` proxy/local, `keystore.toml`, signing/verification) |
| `dtnpm-systemd-deployment-diagnostics` | `skills/dtnpm-systemd-deployment-diagnostics/SKILL.md` | Hardened systemd deployment (`dtnpm.service`), `0600`/`0700` permissions, YunoHost/Nginx reverse proxy, diagnostics (`dtnquery`, `hdy-stats`, `journalctl`) |

## Installation & Discovery

### In an Antigravity Workspace (`.agents/skills.json` or `skills.json`)

This repository provides `skills.json` and `.agents/skills.json` pointing to `skills/`.

1. Reference the path in the `skills.json` of your operating environment:
   ```json
   {
     "entries": [
       { "path": "../skills/skills" }
     ]
   }
   ```
2. Or copy / symlink the desired directories into `~/.gemini/antigravity/skills/`.

## Local Validation & CI

```bash
./scripts/install-hooks.sh
./scripts/validate-skills.sh
```

GitHub CI (`.github/workflows/ci.yml`) exclusively uses actions pinned by full SHA-1 and `step-security/harden-runner` with `permissions: contents: read`.

## License

See [LICENSE.md](LICENSE.md).
