---
name: dtnpm-cla-radio-config
description: >-
  Configures Convergence Layer Adapters (`tcpclv4`, `hqfbp`, `mercury-arq`, `ltp`) inside `dtnpm` or standalone (`dtnpm-cla-hqfbp-kiss`, `dtnpm-cla-mercury-arq`, `dtnpm-cla-ltp-kiss`), including KISS TCP/Serial, Direwolf, Mercury TNC, half-duplex pacing, rate limits, and explicit safety opt-ins (`--require-callsign`, `--allow-unidentified-rf`, `--allow-unauthenticated-udp`, `allow-ip-migration`). Use when wiring radio modems, KISS TNCs, UDP links, or Hamnet TCPCLv4 peers.
---

# Configuration des CLAs Radio & Réseau (`dtnpm-cla-*`)

Cette compétence couvre le paramétrage des quatre familles d'adaptateurs de couche de convergence (CLA) de l'écosystème `dtn-postmaster`, en mode intégré (`[[clas]]` dans `dtnpm`) ou en démon autonome via gRPC/UDS.

## 1. TCPCLv4 sur Hamnet / 44Net / Internet

Dans `config.toml` de `dtnpm` :

```toml
[[clas]]
name = "tcpcl-hamnet"
type = "tcpclv4"
address = "[::]:4556"
# Mode en clair conforme à la réglementation radioamateur (désactiver TLS)
require-tls = false
peers = ["44.27.131.233:4556"]
```

## 2. HQFBP sur KISS (Direwolf, Soundmodem, TNC série)

Adapté aux modems VHF/UHF/HF via KISS TCP (`127.0.0.1:8001` typique sous Direwolf) ou port série (`/dev/ttyUSB0`).

### Configuration intégrée dans `dtnpm` (`config.toml`)

```toml
[[clas]]
name = "kiss-vhf"
type = "hqfbp"
callsign = "F4TEST-1"

[clas.transport]
type = "tcp"
address = "127.0.0.1:8001"
port = 0
```

### Configuration autonome (`dtnpm-cla-hqfbp-kiss`)

Les champs de `Config` (`kebab-case` en TOML/YAML) couvrent :
- `callsign` et `require-callsign` (rejet de `N0CALL` / vide si actif).
- `max-concurrent-senders` (défaut `64`), `max-in-flight-bytes` (défaut 16 Mio), `session-ttl` (défaut `60s`), `sweep-interval` (défaut `5s`).
- `[transport]` :
  - `type = "kiss-tcp"` : `server-addr`, `bind-addr`, `port`, `tx-delay`, `persistence`, `slot-time`, `tx-tail`, `reconnect-initial`, `reconnect-max`, `rate-limit-bytes-per-sec`, `burst-capacity-bytes` (défaut `2048`).
  - `type = "kiss-serial"` : `path` (ex. `"/dev/ttyUSB0"`), `baud-rate` (défaut `9600`), mêmes temporisations KISS et limiteur de débit.
  - `type = "udp"` : `bind-addr`, `peer-addr` (réservé aux flux GNURadio ou bancs locaux).

Démarrage autonome sécurisé :

```bash
dtnpm-cla-hqfbp-kiss \
  --socket /run/dtnpm/grpc.sock \
  --token-file /etc/dtnpm/cla-token.secret \
  --config /etc/dtnpm/hqfbp.toml \
  --require-callsign
```

Exception banc de test / écoute seule (`N0CALL`) : `--allow-unidentified-rf`.

## 3. Mercury Connected ARQ (`dtnpm-cla-mercury-arq`)

Conçu pour les modems HF Mercury exposant un port de contrôle (`127.0.0.1:8300`) et un port de données ARQ (`127.0.0.1:8301`).

Champs principaux (`kebab-case`) :
- `my-callsign`, `require-callsign`
- `control-addr` (`127.0.0.1:8300`), `data-addr` (`127.0.0.1:8301`)
- `bandwidth` (Hz, ex. `500`, `2300`, `2750` ; défaut `2300`)
- `connect-timeout` (`30s`), `grace-period` (`30s`), `idle-timeout`
- `[[peers]]` : `callsign`, `node-id` (optionnel, défaut `dtn://<callsign>/`)

Lancement autonome :

```bash
dtnpm-cla-mercury-arq \
  --socket /run/dtnpm/grpc.sock \
  --token-file /etc/dtnpm/cla-token.secret \
  --config /etc/dtnpm/mercury.toml \
  --require-callsign
```

Option d'opt-in explicite pour banc de test sans indicatif valide : `--allow-unidentified-rf`.

## 4. LTP (RFC 5326 / RFC 7122) sur UDP ou KISS (`dtnpm-cla-ltp-kiss`)

### Paramètres principaux

- `engine-id` : identifiant moteur LTP local (ex. `1001`).
- `[transport]` :
  - `type = "udp"` : `bind-addr` (défaut `0.0.0.0:1113`).
  - `type = "kiss-tcp"` : `server-addr`, `port`, `min-frame-len` (ex. `15` octets minimum requis par Direwolf, `0` par défaut), `rate-limit-bytes-per-sec`, `burst-capacity-bytes` (`2048`), `half-duplex` (`true` sur lien radio simplex), `turnaround-mode` (`"wait-for-report"` ou `"fixed-dwell"`), `turnaround-timeout` (`3s`), `dwell-time` (`500ms`).
- `[[peers]]` : `engine-id`, `address`, `node-id`, `rtt`, `ack-margin`, `allow-ip-migration` (défaut **`false`**).

### Sécurité UDP et opt-in explicite

Par défaut, LTP sur UDP **refuse** les pairs dynamiques non configurés et les migrations d'IP source (`allow-ip-migration = false`) afin de bloquer l'usurpation de routage.

Si l'environnement est déjà sécurisé (tunnel WireGuard/IPsec fermé) ou en laboratoire :
- Par pair : définir `allow-ip-migration = true` dans `[[peers]]` uniquement pour ce pair.
- Mode global non authentifié : passer `--allow-unauthenticated-udp` (ou `DTN_LTP_ALLOW_UNAUTHENTICATED_UDP=1`) au binaire `dtnpm-cla-ltp-kiss` en connaissance du risque.
