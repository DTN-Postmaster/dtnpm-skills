---
name: dtnpm-iam-bpsec-security
description: >-
  Configures identity, access control (DTN-HDY-HAM-IAM), gRPC authentication over Unix Domain Sockets, WebUI proxy/local Argon2id auth, Bearer API tokens, and BPSec RFC 9172 HMAC-SHA256 signing/verification (`keystore.toml`, `VerifyPolicy`). Use when securing `dtnpm`, onboarding operators, rotating tokens, or enabling BPSec bundle authentication.
---

# IAM Security (`DTN-HDY-HAM-IAM`) & BPSec Authentication

This skill details setting up `dtnpm`'s two-tier security model:
1. **Local and gateway IAM (`DTN-HDY-HAM-IAM`)**: WebUI operator authentication (Argon2id or reverse proxy SSO), role-based PBAC (`Admin`, `Operator`), and protection of the local gRPC interface.
2. **BPSec (RFC 9172 — Integrity / BIB HMAC-SHA256)**: end-to-end authentication of BPv7 bundles while remaining compliant with amateur radio regulations (no payload encryption).

## 1. Local WebUI Accounts (`dtnpm user`)

Manage Argon2id users without exposing passwords in shell history or `ps`:

```bash
# Add an administrator or operator reading the password from stdin
printf '%s\n' "$OPERATOR_PASS" | dtnpm user add f4test --role Admin --password-stdin

# Change an existing user's password
printf '%s\n' "$NEW_PASS" | dtnpm user passwd f4test --password-stdin

# List accounts (--data-dir option if using a non-default directory)
dtnpm user list
```

Recognized roles: `Admin` and `Operator`. Avoid the `--password` command-line argument (deprecated).

## 2. API Tokens & gRPC / Unix Domain Socket Protection (`[grpc.auth]`)

### Issue and Revoke Persistent Tokens (`dtnpm token`)

```bash
# Create a token and print the raw secret once (save it with 0600 permissions)
dtnpm token add cli-ops --role Operator > /etc/dtnpm/cli-token.secret
chmod 0600 /etc/dtnpm/cli-token.secret

dtnpm token list
dtnpm token revoke cli-ops
```

### `[grpc.auth]` and `[auth]` Configuration in `config.toml`

Reference from `examples/config-iam-yunohost.toml`:

```toml
[grpc.auth]
enabled = true
socket-path = "/run/dtnpm/grpc.sock"
socket-group = "dtnpm"
socket-mode = "0660"

# Dedicated token (e.g., for a CLA or service)
[[grpc.auth.tokens]]
name = "cla-hqfbp"
role = "Operator"
# Use the stored hash or management via `dtnpm token add` depending on your deployment

[auth]
mode = "proxy"       # "proxy" | "local" | "none"
session-ttl = "12h"
policy-path = "/etc/dtnpm/policy.toml"

[auth.proxy]
trusted-proxies = ["127.0.0.1"]
user-header = "Remote-User"
email-header = "Remote-Email"
role-header = "Remote-Groups"
admin-users = ["f4test"]
default-role = "Operator"

[auth.proxy.callsign-map]
alice = "F4TEST"
```

Always connect CLI clients and external CLAs via `--socket /run/dtnpm/grpc.sock --token-file /etc/dtnpm/cli-token.secret`.

## 3. BPSec Signing and Verification (RFC 9172 / HMAC-SHA256)

### Create a Secure Shared Key File (`0600`)

```bash
umask 0077
openssl rand -hex 32 > ~/.config/dtn/f4test.key
```

### Configure `~/.config/dtn/keystore.toml`

Format verified in `dtnpm-core/src/cli_security.rs` (`KeystoreConfig` / `KeyEntryConfig`):

```toml
[[keys]]
eid = "dtn://f4test/*"
alg = "hmac-sha256"
key_file = "/home/operator/.config/dtn/f4test.key"

[[keys]]
eid = "dtn://f4peer/*"
alg = "hmac-sha256"
key_file = "/home/operator/.config/dtn/f4peer.key"
```

Secure the keystore: `chmod 0600 ~/.config/dtn/keystore.toml`.

### Verification Policies (`--verify-policy`)

- **`strict`**: rejects any unsigned bundle (`Unsigned`), unknown key (`MissingKey`), or invalid signature (`Invalid`). Recommended for exposed services (`dtnbasket`, `dtntrigger`, `template-dtnpm-app`).
- **`warn`**: logs a warning to `stderr` if the signature is missing or fails, but delivers the payload (default in `dtnprint` for progressive interoperability).
- **`ignore`**: disables verification (testbench only).

End-to-end signing + verification example:

```bash
# Strict reception
dtnprint -s incoming --verify-policy strict --keystore ~/.config/dtn/keystore.toml

# Signed transmission
dtnsend -r dtn://f4peer/incoming --sign-key-file ~/.config/dtn/f4test.key --security-source dtn://f4test/ payload.txt
```
