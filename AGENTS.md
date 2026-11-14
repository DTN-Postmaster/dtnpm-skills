# AGENTS.md — DTN Postmaster (`skills`)

Ce document encadre les contributions et l'usage des agents IA sur le catalogue d'exploitation `skills`.

## Règles de conception des `SKILL.md`

1. **Format Antigravity strict** :
   - Chaque skill réside dans `skills/<nom-kebab-case>/SKILL.md`.
   - Le frontmatter YAML est obligatoire et doit déclarer `name` (= nom du dossier) et `description` à la troisième personne expliquant quoi faire et quand activer la skill (<= 1024 caractères).
2. **Exactitude des commandes & configurations (Zero Hallucination)** :
   - Tous les binaires, options CLI, sous-commandes, variables d'environnement et clés TOML/YAML documentés doivent correspondre exactement aux sources de `dtnpm`, `dtnpm-core`, `dtnpm-cla-*` et `dtnpm-app-basket`.
   - En particulier, respecter le comportement réel de chaque outil (`dtnfiles <DIR>` reçoit et enregistre les bundles entrants sur disque ; l'envoi se fait via `dtnsend`).
3. **Sécurité opérationnelle par défaut** :
   - Toujours recommander `--token-file` / `--password-stdin` plutôt que les secrets en argument de ligne de commande.
   - Mettre en avant `VerifyPolicy::Strict` ou `Warn` documenté, le socket UNIX local (`[grpc.auth]`) avec droits `0660`, et les permissions `0600` sur les configurations contenant des secrets.
   - Documenter explicitement que les options dégradées (`--allow-unauthenticated-udp`, `--allow-unidentified-rf`, `allow_ip_migration = true`, `allow_private_ips = true`, `--verify-policy ignore`) ne doivent être activées qu'en connaissance de cause (banc de test ou lien déjà sécurisé).

## Vérification obligatoire avant commit

```bash
./scripts/install-hooks.sh
./scripts/validate-skills.sh
```
