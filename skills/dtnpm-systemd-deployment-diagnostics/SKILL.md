---
name: dtnpm-systemd-deployment-diagnostics
description: >-
  Guides hardened systemd deployment of `dtnpm` (`dtnpm.service`), Linux file permissions (`0600`/`0700`), `systemd-journal` access for `hdy-stats`, YunoHost/Nginx reverse proxy integration, and runtime diagnostics (`dtnquery`, `hdy-stats`, `journalctl`). Use when deploying `dtnpm` in production or troubleshooting node health, routes, and storage.
---

# Hardened systemd Deployment & Diagnostics (`dtnpm`)

This skill brings together Cyber Resilience Act (CRA)-compliant system deployment procedures and day-to-day operation of `dtnpm` on Linux / systemd.

## 1. Hardened systemd Deployment (`dtnpm.service`)

### Dedicated Account and Secure Directory Tree

Create an unprivileged system user and restrict permissions in accordance with `dtnpm/agent_docs/deployment.md`:

```bash
sudo useradd --system --home-dir /var/lib/dtnpm --shell /usr/sbin/nologin dtnpm
sudo install -d -o dtnpm -g dtnpm -m 0700 /var/lib/dtnpm
sudo install -d -o root -g dtnpm -m 0750 /etc/dtnpm
sudo install -d -o dtnpm -g dtnpm -m 0750 /run/dtnpm
```

Mandatory permissions for sensitive files:
- `/etc/dtnpm/config.toml`: `0640` (or `0600` owned by `dtnpm:dtnpm` if the configuration carries secrets).
- `/etc/dtnpm/*.secret`, `keystore.toml`, `.key` files: **`0600`**.

### Hardening Directives in `dtnpm.service`

The unit provided in `dtnpm/dtnpm.service` enforces the following protections:
- `User=dtnpm`, `Group=dtnpm`
- `StateDirectory=dtnpm`, `ConfigurationDirectory=dtnpm`
- `ProtectSystem=strict`, `ProtectHome=true`, `NoNewPrivileges=true`
- `CAP_NET_BIND_SERVICE` only if a port < 1024 is required (otherwise prefer a reverse proxy on `127.0.0.1:8080`).

Lifecycle commands:

```bash
sudo cp dtnpm.service /etc/systemd/system/dtnpm.service
sudo systemctl daemon-reload
sudo systemctl enable --now dtnpm
sudo systemctl status dtnpm
```

## 2. Reverse Proxy Integration (YunoHost / Nginx)

When `[web].address = "127.0.0.1:8080"` is used behind Nginx/SSO:
1. Configure `[auth] mode = "proxy"` and `trusted-proxies = ["127.0.0.1"]` in `/etc/dtnpm/config.toml` (see `examples/config-iam-yunohost.toml`).
2. Forward the `Remote-User`, `Remote-Email`, and `Remote-Groups` headers only from the local reverse proxy (never directly from external clients).
3. Verify that the `dtnpm` backend listens strictly on `127.0.0.1:8080` to prevent any identity header spoofing.

## 3. `hdy-stats` Integration and Journal Reading

To allow `hdy-stats` to query the systemd journal without full `root` privileges:
- Add the service account running `hdy-stats` to the `systemd-journal` system group:
  ```bash
  sudo usermod -aG systemd-journal dtnpm
  ```
- Run collection and view the summary:
  ```bash
  hdy-stats -d /var/lib/dtnpm/hdy-stats.db -r 60 -u dtnpm --show
  ```

Note: the `-u/--journald-unit` flag applies strict validation (rejecting leading dashes and injection metacharacters).

## 4. Production Diagnostics Checklist

In case of an anomaly (stuck bundle, disconnected CLA, full quota):

1. **Daemon status and logs**:
   ```bash
   sudo journalctl -u dtnpm -n 100 --no-pager
   ```
2. **Identity, endpoints, and active peers**:
   ```bash
   dtnquery -c /etc/dtnpm/config.toml info
   dtnquery -c /etc/dtnpm/config.toml eids
   dtnquery -c /etc/dtnpm/config.toml peers
   ```
3. **BPA queue and storage**:
   ```bash
   dtnquery -c /etc/dtnpm/config.toml bundles -v --limit 25
   dtnquery -c /etc/dtnpm/config.toml store --limit 25
   ```
4. **UNIX socket & token permissions**:
   - Verify that `/run/dtnpm/grpc.sock` belongs to the configured group (`socket-mode = "0660"`).
   - Verify that utilities pass `--socket` and `--token-file` readable by the current user.
5. **End-to-end DTN connectivity test**:
   ```bash
   dtnping dtn://<remote-station>/echo -c 3 -w 10s
   ```
