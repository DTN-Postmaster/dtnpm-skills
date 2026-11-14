# Contribuer à `skills`

Merci de contribuer aux guides opérationnels et compétences utilisateurs de `dtn-postmaster`.

## Ajouter ou mettre à jour une skill

1. Créez ou éditez `skills/<nom-skill>/SKILL.md` avec un frontmatter YAML valide (`name` correspondant au dossier, `description` précise à la troisième personne).
2. Vérifiez chaque flag CLI (`--help` ou code source Rust dans `dtnpm/src/bin/`, `dtnpm-app-basket/src/`, `dtnpm-cla-*/src/`) et chaque clé TOML/YAML (`config.example.toml`, `examples/config-iam-yunohost.toml`).
3. N'introduisez aucun vrai jeton, mot de passe ou indicatif privé réel ; utilisez uniquement `F4TEST` ou `N0CALL` dans les exemples.
4. Exécutez `./scripts/validate-skills.sh` avant tout commit.
