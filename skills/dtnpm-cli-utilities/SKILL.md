---
name: dtnpm-cli-utilities
description: >-
  Reference guide for operating DTN Postmaster CLI utilities (`dtnsend`, `dtnprint`, `dtnping`, `dtnfiles`, `dtnquery`, `dtntrigger`, `dtnforward`, `dtnbib`, `hdy-stats`, `dtnbasket`, `dtnbasket-cli`) with exact flags, Unix socket / `--token-file` authentication, and BPSec signing/verification options. Use when invoking, scripting, or troubleshooting `dtnpm` command-line tools.
---

# DTN Postmaster CLI Utilities

All client utilities connect to the `dtnpm` daemon via `--socket <PATH>` (`DTN_HDY_SOCKET`) or localhost gRPC, and preferably authenticate via `--token-file <FILE>` (`DTN_HDY_TOKEN_FILE`). Avoid passing `--token` in cleartext in shared scripts.

## 1. Sending, Receiving, and Basic Diagnostics

### `dtnsend` — Send a bundle from a file or `stdin`

```bash
# Send file contents with BPSec HMAC-SHA256 signature
dtnsend --socket /run/dtnpm/grpc.sock --token-file /etc/dtnpm/token.secret \
  -r dtn://f4peer/incoming -s incoming --sign-key-file /etc/dtnpm/hmac.key \
  message.txt

# Send from stdin with custom lifetime (seconds) or dry run (-D)
printf "Test CQ" | dtnsend -r dtn://f4peer/incoming -l 7200
dtnsend -r dtn://f4peer/incoming -D message.txt
```

Verified options: `-r <EID>`, `[INFILE]`, `-s/--sender`, `-l/--lifetime` (`3600`), `-D/--dryrun`, `--sign-key`, `--sign-key-file`, `--security-source`, `-p/--port`, `-6`, `--token-file`, `--socket`.

### `dtnprint` — Display bundles received on a service

```bash
dtnprint --socket /run/dtnpm/grpc.sock --token-file /etc/dtnpm/token.secret \
  -s incoming --verify-policy strict --keystore ~/.config/dtn/keystore.toml
```

Verified options: `-s/--service` (default `"incoming"`), `--keystore`, `--verify-key`, `--verify-key-file`, `--verify-policy <strict|warn|ignore>` (default `warn`), `-p`, `-6`, `--token-file`, `--socket`.

### `dtnfiles` — Receive and save incoming bundles to disk

> [!IMPORTANT]
> `dtnfiles` is a **receiver** that saves each received bundle into a secure local directory (with path traversal sanitization) — it is not a sending command. To send a file, use `dtnsend`.

```bash
dtnfiles /var/lib/dtnpm/inbox \
  --socket /run/dtnpm/grpc.sock --token-file /etc/dtnpm/token.secret \
  -s files --max-size-mb 100 -m --verify-policy strict
```

Verified arguments: `<DIR>` (required positional), `-s/--service` (`"files"`), `-m/--metadata`, `--max-size-mb` (`100`), `--keystore`, `--verify-policy`, `--token-file`, `--socket`.

### `dtnping` — Measure reachability and latency to a DTN node

```bash
dtnping dtn://f4peer/echo -c 5 -i 2s -w 10s -s 64 --keystore ~/.config/dtn/keystore.toml
```

Verified options: `<destination>`, `-c/--count`, `-i/--interval` (`1s`), `-s/--size`, `-w/--timeout`, `-W/--wait` (`10s`), `-q/--quiet`, `-t/--ttl`, `--lifetime`, `-S/--source`, `--keystore`, `--verify-*`, `--sign-*`.

## 2. Automation, Routing, and Inspection

### `dtnquery` — Inspect the BPA and bundle store

```bash
dtnquery -c /etc/dtnpm/config.toml eids
dtnquery -c /etc/dtnpm/config.toml peers
dtnquery -c /etc/dtnpm/config.toml bundles -v --limit 50
dtnquery -c /etc/dtnpm/config.toml store --limit 100
dtnquery -c /etc/dtnpm/config.toml info
dtnquery -c /etc/dtnpm/config.toml nodeid
```

Additional filters on `bundles`: `-d`, `-a <addr>`, `--limit <N>`.

### `dtntrigger` — Trigger a local executable upon bundle reception

Executes the target command without invoking a shell (`param1=source`, `param2=payload_file`):

```bash
dtntrigger -e trigger -c /usr/local/bin/on_bundle.sh \
  --verify-policy strict --keystore ~/.config/dtn/keystore.toml \
  --socket /run/dtnpm/grpc.sock --token-file /etc/dtnpm/token.secret
```

Verified options: `-e/--endpoint`, `-c/--command` (default `"echo"`), `--print`, `--keystore`, `--verify-policy`, `--token-file`, `--socket`.

### `dtnforward` & `dtnbib` — Relay and BIBE encapsulation gateway

- **`dtnforward`**: relays bundles received on `-s/--service` (`"incoming"`) to `-t/--target <EID>` with loop guard `--max-depth` (`10`), `--lifetime-default` (`3600`), `-S/--sender`, `--raw`, and BPSec options (`--keystore`, `--verify-policy`, `--sign-key*`, `--security-source`).
- **`dtnbib`**: BIBE encapsulation service (`-s/--service` default `"bibe"`, `-a/--alias <src=dst,...>`, `--no-transit`, BPSec and gRPC/UDS options).

### `hdy-stats` — Telemetry and statistics collector

```bash
hdy-stats -d /var/lib/dtnpm/hdy-stats.db -r 60 -u dtnpm --show
```

Verified options: `-d/--database` (`hdy-stats.db`), `-r/--retention` (`60`), `-f/--log-file`, `-u/--journald-unit` (`hardy-bpa`), `--sudo`, `-s/--service` (`stats`), `-S/--show`, `--allow-third-party-report-to`, `--token-file`, `--socket`.

## 3. `dtnpm-app-basket` Service (`dtnbasket` and `dtnbasket-cli`)

- **`dtnbasket` daemon**: `-c/--config` (`DTNBASKET_CONFIG`), `-s/--service` (default `dtnbasket`), `--keystore`, `--verify-*`, `--sign-*`, `--token-file`, `--socket`. In the TOML configuration, `[security]` blocks private IPs and *third-party reply-to* by default (`allow_private_ips = false`, `allow_third_party_reply_to = false`), while `allowed_dirs` and `mappings` restrict accessible local resources.
- **`dtnbasket-cli` client**: `-s/--sender` (`basket_client`), `-r/--receiver`, `--get <URI>`, `--search <URI>`, `--list <URI>`, `-f/--request-file`, `--req-id`, `--reply-to`, `--sign-*`, `--token-file`, `--socket`.
