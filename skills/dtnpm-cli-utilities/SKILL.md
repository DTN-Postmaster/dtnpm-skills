---
name: dtnpm-cli-utilities
description: >-
  Reference guide for operating DTN Postmaster CLI utilities (`dtnsend`, `dtnprint`, `dtnping`, `dtnfiles`, `dtnquery`, `dtntrigger`, `dtnforward`, `dtnbib`, `hdy-stats`, `dtnbasket`, `dtnbasket-cli`) with exact flags, Unix socket / `--token-file` authentication, and BPSec signing/verification options. Use when invoking, scripting, or troubleshooting `dtnpm` command-line tools.
---

# Utilitaires CLI de DTN Postmaster

Tous les utilitaires clients se connectent au daemon `dtnpm` via `--socket <CHEMIN>` (`DTN_HDY_SOCKET`) ou gRPC localhost, et s'authentifient de préférence via `--token-file <FICHIER>` (`DTN_HDY_TOKEN_FILE`). Éviter `--token` en clair dans les scripts partagés.

## 1. Envoi, réception et diagnostic de base

### `dtnsend` — Envoyer un bundle depuis un fichier ou `stdin`

```bash
# Envoyer le contenu d'un fichier avec signature BPSec HMAC-SHA256
dtnsend --socket /run/dtnpm/grpc.sock --token-file /etc/dtnpm/token.secret \
  -r dtn://f4peer/incoming -s incoming --sign-key-file /etc/dtnpm/hmac.key \
  message.txt

# Envoyer depuis stdin avec durée de vie personnalisée (secondes) ou essai à blanc (-D)
printf "Test CQ" | dtnsend -r dtn://f4peer/incoming -l 7200
dtnsend -r dtn://f4peer/incoming -D message.txt
```

Options vérifiées : `-r <EID>`, `[INFILE]`, `-s/--sender`, `-l/--lifetime` (`3600`), `-D/--dryrun`, `--sign-key`, `--sign-key-file`, `--security-source`, `-p/--port`, `-6`, `--token-file`, `--socket`.

### `dtnprint` — Afficher les bundles reçus sur un service

```bash
dtnprint --socket /run/dtnpm/grpc.sock --token-file /etc/dtnpm/token.secret \
  -s incoming --verify-policy strict --keystore ~/.config/dtn/keystore.toml
```

Options vérifiées : `-s/--service` (défaut `"incoming"`), `--keystore`, `--verify-key`, `--verify-key-file`, `--verify-policy <strict|warn|ignore>` (défaut `warn`), `-p`, `-6`, `--token-file`, `--socket`.

### `dtnfiles` — Réceptionner et enregistrer les bundles entrants sur disque

> [!IMPORTANT]
> `dtnfiles` est un **récepteur** qui sauvegarde chaque bundle reçu dans un répertoire local sécurisé (avec assainissement contre le traversal) — ce n'est pas une commande d'envoi. Pour envoyer un fichier, utiliser `dtnsend`.

```bash
dtnfiles /var/lib/dtnpm/inbox \
  --socket /run/dtnpm/grpc.sock --token-file /etc/dtnpm/token.secret \
  -s files --max-size-mb 100 -m --verify-policy strict
```

Arguments vérifiés : `<DIR>` (positionnel requis), `-s/--service` (`"files"`), `-m/--metadata`, `--max-size-mb` (`100`), `--keystore`, `--verify-policy`, `--token-file`, `--socket`.

### `dtnping` — Mesurer l'accessibilité et la latence d'un nœud DTN

```bash
dtnping dtn://f4peer/echo -c 5 -i 2s -w 10s -s 64 --keystore ~/.config/dtn/keystore.toml
```

Options vérifiées : `<destination>`, `-c/--count`, `-i/--interval` (`1s`), `-s/--size`, `-w/--timeout`, `-W/--wait` (`10s`), `-q/--quiet`, `-t/--ttl`, `--lifetime`, `-S/--source`, `--keystore`, `--verify-*`, `--sign-*`.

## 2. Automatisation, routage et inspection

### `dtnquery` — Inspecter le BPA et le store de bundles

```bash
dtnquery -c /etc/dtnpm/config.toml eids
dtnquery -c /etc/dtnpm/config.toml peers
dtnquery -c /etc/dtnpm/config.toml bundles -v --limit 50
dtnquery -c /etc/dtnpm/config.toml store --limit 100
dtnquery -c /etc/dtnpm/config.toml info
dtnquery -c /etc/dtnpm/config.toml nodeid
```

Filtres supplémentaires sur `bundles` : `-d`, `-a <addr>`, `--limit <N>`.

### `dtntrigger` — Déclencher un exécutable local à la réception d'un bundle

Exécute la commande cible sans invoquer de shell (`param1=source`, `param2=payload_file`) :

```bash
dtntrigger -e trigger -c /usr/local/bin/on_bundle.sh \
  --verify-policy strict --keystore ~/.config/dtn/keystore.toml \
  --socket /run/dtnpm/grpc.sock --token-file /etc/dtnpm/token.secret
```

Options vérifiées : `-e/--endpoint`, `-c/--command` (défaut `"echo"`), `--print`, `--keystore`, `--verify-policy`, `--token-file`, `--socket`.

### `dtnforward` & `dtnbib` — Relais et passerelle d'encapsulation BIBE

- **`dtnforward`** : relaye les bundles reçus sur `-s/--service` (`"incoming"`) vers `-t/--target <EID>` avec garde anti-boucle `--max-depth` (`10`), `--lifetime-default` (`3600`), `-S/--sender`, `--raw`, options BPSec (`--keystore`, `--verify-policy`, `--sign-key*`, `--security-source`).
- **`dtnbib`** : service d'encapsulation BIBE (`-s/--service` défaut `"bibe"`, `-a/--alias <src=dst,...>`, `--no-transit`, options BPSec et gRPC/UDS).

### `hdy-stats` — Collecteur de télémétrie et statistiques

```bash
hdy-stats -d /var/lib/dtnpm/hdy-stats.db -r 60 -u dtnpm --show
```

Options vérifiées : `-d/--database` (`hdy-stats.db`), `-r/--retention` (`60`), `-f/--log-file`, `-u/--journald-unit` (`hardy-bpa`), `--sudo`, `-s/--service` (`stats`), `-S/--show`, `--allow-third-party-report-to`, `--token-file`, `--socket`.

## 3. Service `dtnpm-app-basket` (`dtnbasket` et `dtnbasket-cli`)

- **Démon `dtnbasket`** : `-c/--config` (`DTNBASKET_CONFIG`), `-s/--service` (défaut `dtnbasket`), `--keystore`, `--verify-*`, `--sign-*`, `--token-file`, `--socket`. Dans la configuration TOML, `[security]` bloque par défaut les IP privées et le *third-party reply-to* (`allow_private_ips = false`, `allow_third_party_reply_to = false`), tandis que `allowed_dirs` et `mappings` restreignent les ressources locales accessibles.
- **Client `dtnbasket-cli`** : `-s/--sender` (`basket_client`), `-r/--receiver`, `--get <URI>`, `--search <URI>`, `--list <URI>`, `-f/--request-file`, `--req-id`, `--reply-to`, `--sign-*`, `--token-file`, `--socket`.
