---
name: dtnpm-station-setup
description: >-
  Guides operators through setting up and configuring a DTN Postmaster (`dtnpm`) station: Cargo build profiles (`emcom`, `amsat`, `node-headless`, `webui-remote`), `dtnpm init` and `generate-config`, TOML/YAML station EIDs, SQLite storage and 4-tier LRU quotas, static routes, RFC 9171 validity toggles, and WebUI access. Use when initializing or configuring a DTN Postmaster node.
---

# Setting Up and Configuring a `dtnpm` Station

This skill guides the complete commissioning of a BPv7 DTN station with `dtnpm` (full BPA node, lightweight AMSAT station, headless node, or remote WebUI).

## 1. Choose the Compilation Profile (`dtnpm`)

From the `dtnpm` repository, select the appropriate profile via Cargo features:

| Profile / Feature | Use Case | Build Command |
| --- | --- | --- |
| Default (`emcom`) | Full station (BPA node + `ltp`/`hqfbp`/`mercury` CLAs + `chat`/`files`/`senml` WebUI + CLI utilities) | `cargo build --release` |
| `amsat` | Lightweight satellite / AMSAT configuration (`node`, `cli`, `cla-ltp`, `cla-hqfbp`) | `cargo build --release --no-default-features --features amsat` |
| `node-headless` | BPA daemon + CLAs without WebUI or interactive CLI (`node`, CLAs) | `cargo build --release --no-default-features --features node-headless` |
| `webui-remote` | Standalone WebUI connected to a remote BPA over gRPC/UDS | `cargo build --release --no-default-features --features webui-remote` |

## 2. Generate the Initial Configuration

Generate an annotated skeleton or launch the interactive wizard:

```bash
# Generate an example configuration (YAML)
dtnpm generate-config > config.yaml

# Or run the interactive wizard to a specific file
dtnpm init -o /etc/dtnpm/config.toml
chmod 0600 /etc/dtnpm/config.toml
```

Both **TOML** (`.toml`) and **YAML** (`.yaml` / `.yml`) formats are transparently recognized by `dtnpm`.

## 3. Essential Configuration File Parameters (`config.toml`)

```toml
# Amateur radio identity and RFC 9171 primary EID
callsign = "F4TEST"
node-id = "dtn://f4test/"
admin-endpoints = []

[storage]
data-dir = "/var/lib/dtnpm"
in-memory = false

[application.quota]
# Global application disk quota (4-tier LRU cascade protecting pinned items)
max-storage = "300MB"

[grpc]
enabled = true
address = "127.0.0.1:50051" # Or UNIX socket via [grpc.auth]

[web]
enabled = true
address = "127.0.0.1:8080"

[rfc9171-validity]
# Set to false for maximum interoperability (e.g., dtn7-rs, dtn-android-messenger)
primary-block-integrity = false
bundle-age-required = false
```

### Static Routing (`[[routes]]`)

Map EID patterns (`pattern`) to names declared in `[[clas]]`:

```toml
[[routes]]
pattern = "dtn://*/*"
cla = "tcpcl-hamnet"
```

More specific routes (e.g., `"dtn://f4peer/*"`) take precedence over the global wildcard `"dtn://*/*"`.

## 4. Start the Station

```bash
# Standard mode with TOML/YAML configuration
dtnpm -c /etc/dtnpm/config.toml

# Ad-hoc EID override or ephemeral in-memory mode (testbench)
dtnpm --in-memory -n "dtn://f4test-lab/"

# Connect to a remote BPA with file-based authentication
dtnpm --remote-bpa http://127.0.0.1:50051 --token-file /etc/dtnpm/token.secret
# Or via local Unix Domain Socket
dtnpm --socket /run/dtnpm/grpc.sock --token-file /etc/dtnpm/token.secret
```

## 5. Regulatory & Initial Security Reminders

- On amateur radio frequencies, **do not encrypt payloads** (`require-tls = false` in TCPCLv4 over RF/Hamnet if required by the local license); prefer **BPSec BIB HMAC-SHA256** signing/authentication only.
- By default, restrict `[web].address` and `[grpc].address` to `127.0.0.1` or a Unix Domain Socket, and enable `[auth]` + `[grpc.auth]` as soon as another user or service accesses the host.
