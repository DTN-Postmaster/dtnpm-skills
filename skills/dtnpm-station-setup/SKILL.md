---
name: dtnpm-station-setup
description: >-
  Guides operators through setting up and configuring a DTN Postmaster (`dtnpm`) station: Cargo build profiles (`emcom`, `amsat`, `node-headless`, `webui-remote`), `dtnpm init` and `generate-config`, TOML/YAML station EIDs, SQLite storage and 4-tier LRU quotas, static routes, RFC 9171 validity toggles, and WebUI access. Use when initializing or configuring a DTN Postmaster node.
---

# Installation et configuration d'une station `dtnpm`

Cette compétence guide la mise en service complète d'une station DTN BPv7 avec `dtnpm` (nœud BPA complet, station légère AMSAT, nœud headless ou interface Web distante).

## 1. Choisir le profil de compilation (`dtnpm`)

Depuis le dépôt `dtnpm`, sélectionner le profil adapté via les features Cargo :

| Profil / Feature | Cas d'usage | Commande de build |
| --- | --- | --- |
| Défaut (`emcom`) | Station complète (nœud BPA + CLAs `ltp`/`hqfbp`/`mercury` + WebUI `chat`/`files`/`senml` + utilitaires CLI) | `cargo build --release` |
| `amsat` | Configuration légère satellite / AMSAT (`node`, `cli`, `cla-ltp`, `cla-hqfbp`) | `cargo build --release --no-default-features --features amsat` |
| `node-headless` | Démon BPA + CLAs sans interface Web ni CLI interactive (`node`, CLAs) | `cargo build --release --no-default-features --features node-headless` |
| `webui-remote` | Interface WebUI autonome connectée à un BPA distant via gRPC/UDS | `cargo build --release --no-default-features --features webui-remote` |

## 2. Générer la configuration initiale

Générer un squelette annoté ou lancer l'assistant interactif :

```bash
# Générer un exemple de configuration (YAML)
dtnpm generate-config > config.yaml

# Ou lancer l'assistant interactif vers un fichier précis
dtnpm init -o /etc/dtnpm/config.toml
chmod 0600 /etc/dtnpm/config.toml
```

Les formats **TOML** (`.toml`) et **YAML** (`.yaml` / `.yml`) sont tous deux reconnus de façon transparente par `dtnpm`.

## 3. Paramètres essentiels du fichier de configuration (`config.toml`)

```toml
# Identité radioamateur et EID primaire RFC 9171
callsign = "F4TEST"
node-id = "dtn://f4test/"
admin-endpoints = []

[storage]
data-dir = "/var/lib/dtnpm"
in-memory = false

[application.quota]
# Quota disque global applicatif (cascade LRU à 4 niveaux protégeant les éléments épinglés)
max-storage = "300MB"

[grpc]
enabled = true
address = "127.0.0.1:50051" # Ou socket UNIX via [grpc.auth]

[web]
enabled = true
address = "127.0.0.1:8080"

[rfc9171-validity]
# Mettre à false pour l'interopérabilité maximale (ex. dtn7-rs, dtn-android-messenger)
primary-block-integrity = false
bundle-age-required = false
```

### Routage statique (`[[routes]]`)

Associer des motifs EID (`pattern`) aux noms déclarés dans `[[clas]]` :

```toml
[[routes]]
pattern = "dtn://*/*"
cla = "tcpcl-hamnet"
```

Les routes les plus spécifiques (ex. `"dtn://f4peer/*"`) priment sur le joker global `"dtn://*/*"`.

## 4. Démarrer la station

```bash
# Mode standard avec configuration TOML/YAML
dtnpm -c /etc/dtnpm/config.toml

# Surcharge ponctuelle de l'EID ou mode mémoire éphémère (banc de test)
dtnpm --in-memory -n "dtn://f4test-lab/"

# Connexion vers un BPA distant avec authentification sécurisée par fichier
dtnpm --remote-bpa http://127.0.0.1:50051 --token-file /etc/dtnpm/token.secret
# Ou via Unix Domain Socket local
dtnpm --socket /run/dtnpm/grpc.sock --token-file /etc/dtnpm/token.secret
```

## 5. Rappels réglementaires et sécurité initiale

- Sur les fréquences radioamateurs, **ne pas chiffrer les charges utiles** (`require-tls = false` en TCPCLv4 sur RF/Hamnet si requis par la licence locale) ; privilégier uniquement la signature/authentification **BPSec BIB HMAC-SHA256**.
- Par défaut, limiter `[web].address` et `[grpc].address` à `127.0.0.1` ou à un Unix Domain Socket, et activer `[auth]` + `[grpc.auth]` dès qu'un autre utilisateur ou service accède à l'hôte.
