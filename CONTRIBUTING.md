# Contributing to `skills`

Thank you for contributing to the operational guides and user skills for `dtn-postmaster`.

## Adding or Updating a Skill

1. Create or edit `skills/<skill-name>/SKILL.md` with valid YAML frontmatter (`name` matching the directory, accurate third-person `description`).
2. Verify every CLI flag (`--help` or Rust source code in `dtnpm/src/bin/`, `dtnpm-app-basket/src/`, `dtnpm-cla-*/src/`) and every TOML/YAML key (`config.example.toml`, `examples/config-iam-yunohost.toml`).
3. Never introduce real tokens, passwords, or real private callsigns; only use `F4TEST` or `N0CALL` in examples.
4. Run `./scripts/validate-skills.sh` before any commit.
