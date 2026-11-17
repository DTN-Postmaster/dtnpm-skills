# AGENTS.md — DTN Postmaster (`skills`)

This document governs contributions and the use of AI agents on the `skills` operational catalog.

## Documentation Language

English must be used as the language for all documentation, unless the file is suffixed with `.fr.md` or another language is explicitly requested in the prompt.

## `SKILL.md` Design Rules

1. **Strict Antigravity format**:
   - Each skill lives in `skills/<kebab-case-name>/SKILL.md`.
   - YAML frontmatter is mandatory and must declare `name` (= directory name) and `description` in the third person explaining what to do and when to activate the skill (<= 1024 characters).
2. **Command & configuration accuracy (Zero Hallucination)**:
   - All documented binaries, CLI options, subcommands, environment variables, and TOML/YAML keys must match the sources of `dtnpm`, `dtnpm-core`, `dtnpm-cla-*`, and `dtnpm-app-basket` verbatim.
   - In particular, respect the actual behavior of each tool (`dtnfiles <DIR>` receives and writes incoming bundles to disk; sending is done via `dtnsend`).
3. **Operational security by default**:
   - Always recommend `--token-file` / `--password-stdin` over secrets passed as command-line arguments.
   - Highlight `VerifyPolicy::Strict` or documented `Warn`, the local UNIX socket (`[grpc.auth]`) with `0660` permissions, and `0600` permissions on configurations containing secrets.
   - Explicitly document that degraded options (`--allow-unauthenticated-udp`, `--allow-unidentified-rf`, `allow_ip_migration = true`, `allow_private_ips = true`, `--verify-policy ignore`) must only be enabled knowingly (testbench or already-secured link).

## Mandatory Verification Before Commit

```bash
./scripts/install-hooks.sh
./scripts/validate-skills.sh
```
