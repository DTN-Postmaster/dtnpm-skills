# Security Policy — `skills`

## Documentation Best Practices

The skills in this repository describe system, network, systemd, and cryptographic security (IAM + BPSec) configurations. Incorrect documentation can lead an operator to expose their DTN node without authentication or leak secrets in shell history or `/proc`.

Any contribution must therefore:
- Prefer `--token-file`, `DTN_HDY_TOKEN_FILE`, `--password-stdin`, and authenticated Unix Domain Sockets.
- Clearly highlight the implications of dangerous opt-in flags (`--allow-unauthenticated-udp`, `--allow-unidentified-rf`, `--verify-policy ignore`, `--allow-third-party-report-to`, `allow_private_ips`).
- Permanent amateur radio reminder: payload encryption is prohibited on amateur frequencies; only **authentication / integrity (BPSec BIB HMAC-SHA256)** and cleartext traffic are compliant on amateur RF.

## Reporting a Security Issue

Contact `security@dtn-postmaster.org` or open a private *Security Advisory* on GitHub for any dangerous recommendation or detected vulnerability.
