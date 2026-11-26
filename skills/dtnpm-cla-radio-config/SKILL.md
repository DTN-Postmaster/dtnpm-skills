---
name: dtnpm-cla-radio-config
description: >-
  Configures Convergence Layer Adapters (`tcpclv4`, `hqfbp`, `mercury-arq`, `ltp`, `file-cla`) inside `dtnpm` or standalone (`dtnpm-cla-hqfbp-kiss`, `dtnpm-cla-mercury-arq`, `dtnpm-cla-ltp-kiss`), including KISS TCP/Serial, Direwolf, Mercury TNC, `esp-pdu-tnc` (ESP-NOW 250-byte MTU, RaptorQ vs Reed-Solomon encoding selection), half-duplex pacing, rate limits, file-based sneakernet/external-tool spooling, and explicit safety opt-ins (`--require-callsign`, `--allow-unidentified-rf`, `--allow-unauthenticated-udp`, `allow-ip-migration`). Use when wiring radio modems, KISS TNCs, UDP links, Hamnet TCPCLv4 peers, or filesystem bundle bridges.
---

# Radio & Network CLA Configuration (`dtnpm-cla-*`)

This skill covers configuring the five Convergence Layer Adapter (CLA) families in the `dtn-postmaster` ecosystem, either in integrated mode (`[[clas]]` inside `dtnpm`) or as a standalone daemon over gRPC/UDS.

## 1. TCPCLv4 over Hamnet / 44Net / Internet

In `dtnpm`'s `config.toml`:

```toml
[[clas]]
name = "tcpcl-hamnet"
type = "tcpclv4"
address = "[::]:4556"
# Cleartext mode compliant with amateur radio regulations (disable TLS)
require-tls = false
peers = ["44.27.131.233:4556"]
```

## 2. HQFBP over KISS (Direwolf, Soundmodem, ESP-NOW, Serial TNC)

Suitable for VHF/UHF/HF modems via KISS TCP (`127.0.0.1:8001` under Direwolf, `127.0.0.1:18001` under `esp-pdu-tnc`) or serial port (`/dev/ttyUSB0`).

### Integrated Configuration in `dtnpm` (`config.toml`)

```toml
[[clas]]
name = "kiss-vhf"
type = "hqfbp"
callsign = "F4TEST-1"

[clas.transport]
type = "kiss-tcp"
server-addr = "127.0.0.1:8001"
port = 0
```

### Channel-Matched HQFBP Encoding Pipelines

Select the encoding pipeline to match the physical link's error behavior and MTU:

- **Direwolf / AFSK / G3RUH (bit-error channel)**:
  Use chunking plus Reed-Solomon error correction:
  `hailing-encodings = "gzip, chunk(150), crc32, h, rs(255, 233)"`
  `default-bundle-encodings = "gzip, chunk(150), crc32, h, rs(255, 233)"`
- **ESP-NOW via `esp-pdu-tnc` (250-byte max PDU, hardware 802.11b CRC-32 FCS packet-erasure channel over 115200-baud UART)**:
  **Do not** use `rs(255, 233)` — 255-byte codewords exceed the 250-byte ESP-NOW MTU (causing silent drops in `esp-pdu-tnc`), and intra-frame RS cannot recover frames already discarded by the 802.11 MAC FCS. Instead use RaptorQ erasure coding with UART rate pacing (see `dtnpm/examples/stata-esp-hqfbp.yaml` / `statb-esp-hqfbp.yaml`):
  - `hailing-encodings = "h"`
  - `default-bundle-encodings = "gzip, crc32, rq(dlen, 140, 2), h"` (`140`-byte symbols + 4 B PayloadId + ~65–80 B CBOR header fit inside 250 bytes, with 2 repair symbols and self-describing headers)
  - `rate-limit-bytes-per-sec = 4000`, `burst-capacity-bytes = 512`

### Standalone Configuration (`dtnpm-cla-hqfbp-kiss`)

`Config` fields (`kebab-case` in TOML/YAML) cover:
- `callsign` and `require-callsign` (rejects `N0CALL` / empty if enabled).
- `max-concurrent-senders` (default `64`), `max-in-flight-bytes` (default 16 MiB), `session-ttl` (default `60s`), `sweep-interval` (default `5s`).
- `[transport]`:
  - `type = "kiss-tcp"`: `server-addr`, `bind-addr`, `port`, `tx-delay`, `persistence`, `slot-time`, `tx-tail`, `reconnect-initial`, `reconnect-max`, `rate-limit-bytes-per-sec`, `burst-capacity-bytes` (default `2048`).
  - `type = "kiss-serial"`: `path` (e.g., `"/dev/ttyUSB0"`), `baud-rate` (default `9600`), same KISS timings and rate limiter.
  - `type = "udp"`: `bind-addr`, `peer-addr` (reserved for GNURadio streams or local testbenches).

Secure standalone startup:

```bash
dtnpm-cla-hqfbp-kiss \
  --socket /run/dtnpm/grpc.sock \
  --token-file /etc/dtnpm/cla-token.secret \
  --config /etc/dtnpm/hqfbp.toml \
  --require-callsign
```

Testbench / listen-only exception (`N0CALL`): `--allow-unidentified-rf`.

## 3. Mercury Connected ARQ (`dtnpm-cla-mercury-arq`)

Designed for Mercury HF modems exposing a control port (`127.0.0.1:8300`) and an ARQ data port (`127.0.0.1:8301`).

Main fields (`kebab-case`):
- `my-callsign`, `require-callsign`
- `control-addr` (`127.0.0.1:8300`), `data-addr` (`127.0.0.1:8301`)
- `bandwidth` (Hz, e.g., `500`, `2300`, `2750`; default `2300`)
- `connect-timeout` (`30s`), `grace-period` (`30s`), `idle-timeout`
- `[[peers]]`: `callsign`, `node-id` (optional, default `dtn://<callsign>/`)

Standalone launch:

```bash
dtnpm-cla-mercury-arq \
  --socket /run/dtnpm/grpc.sock \
  --token-file /etc/dtnpm/cla-token.secret \
  --config /etc/dtnpm/mercury.toml \
  --require-callsign
```

Explicit opt-in option for testbench without a valid callsign: `--allow-unidentified-rf`.

## 4. LTP (RFC 5326 / RFC 7122) over UDP or KISS (`dtnpm-cla-ltp-kiss`)

### Main Parameters

- `engine-id`: local LTP engine identifier (e.g., `1001`).
- `max-segment-payload-len`: maximum LTP data segment payload in bytes (default `1400`). Must be lowered to match link MTU: e.g., `200` for 250-byte ESP-NOW `esp-pdu-tnc` links (`dtnpm/examples/stata-esp-ltp.yaml` / `statb-esp-ltp.yaml`), or `80` for narrowband Mercury HF links (`stata-ltp.yaml`).
- `[transport]`:
  - `type = "udp"`: `bind-addr` (default `0.0.0.0:1113`).
  - `type = "kiss-tcp"`: `server-addr`, `port`, `min-frame-len` (e.g., `15` bytes minimum required by Direwolf, `0` by default), `rate-limit-bytes-per-sec`, `burst-capacity-bytes` (`2048`), `half-duplex` (`true` on simplex radio link), `turnaround-mode` (`"wait-for-report"` or `"fixed-dwell"`), `turnaround-timeout` (`3s`), `dwell-time` (`500ms`).
- `[[peers]]`: `engine-id`, `address`, `node-id`, `rtt`, `ack-margin`, `allow-ip-migration` (default **`false`**).

### UDP Security and Explicit Opt-In

By default, LTP over UDP **rejects** unconfigured dynamic peers and source IP migrations (`allow-ip-migration = false`) in order to block routing spoofing.

If the environment is already secured (closed WireGuard/IPsec tunnel) or in a lab:
- Per peer: set `allow-ip-migration = true` in `[[peers]]` only for that peer.
- Global unauthenticated mode: pass `--allow-unauthenticated-udp` (or `DTN_LTP_ALLOW_UNAUTHENTICATED_UDP=1`) to the `dtnpm-cla-ltp-kiss` binary with full awareness of the risk.

## 5. File-Based CLA (`file-cla` — Sneakernet / USB / External Tools)

Integrated in `dtnpm` via the `cla-file` feature (`desktop`, `amsat`, `node-headless`, and `minimal` profiles). You can instantiate multiple `file-cla` entries with distinct `name`, directories, and APRS `aliases`.

```yaml
clas:
  - name: "file-usb"
    type: "file-cla"          # or "file"
    aliases: ["USB"]          # automatic transport tag "FILE" is also added
    max-bundle-size: "64MB"   # optional size cap (default: 64MB; prevents OOM on oversized files)
    outbox: "~/dtn-spool/usb/outbox"   # watched for incoming raw BPv7 files (deleted after dispatch)
    peers:
      "dtn://f4usb/": "~/dtn-spool/usb/inbox"  # dtnpm writes forwarded bundles here

  - name: "file-ext-tool"
    type: "file-cla"
    aliases: ["EXT"]
    outbox: "/var/lib/dtnpm/spool/ext/outbox"
    peers:
      "dtn://f4ext/": "/var/lib/dtnpm/spool/ext/inbox"
```

- `~/` is expanded against `$HOME`. Missing `outbox` and `peers` directories are created eagerly at startup with `0700` permissions on Unix (`dtnpm` warns if an existing spool directory is group/world-accessible); if any path cannot be created or accessed, `dtnpm` fails startup with an error.
- **Systemd (`dtnpm.service`)**: `dtnpm.service` enforces `ProtectSystem=strict` and `ProtectHome=true`, which block writes to `~/...` and `/var/spool/...`. When running under systemd, place spool directories under `/var/lib/dtnpm/spool/...` (`StateDirectory=dtnpm`) or explicitly whitelist custom paths via `ReadWritePaths=` in a unit drop-in.
- **Path & peer safety guards**: Each peer within a `file-cla` instance must have a distinct `inbox` directory, and `outbox` cannot be empty, `"."`, a station `data-dir`/config directory, or any CLA's `refused/` quarantine directory.
- **Atomic spooling**: External tools producing bundles into `outbox` should write to a temporary file outside `outbox` and atomically `rename(2)` it into `outbox` to avoid partial reads. Conversely, `hardy-file-cla` currently writes forwarded bundles to `inbox` via `tokio::fs::write` (non-atomic), so external tools consuming from `inbox` over slow media should allow writes to settle before reading.
- APRS routing: `--aprs-path USB1-1` routes only to `file-usb`, `--aprs-path EXT1-1` routes only to `file-ext-tool`, and `--aprs-path FILE1-1` reaches all `file-cla` instances.
