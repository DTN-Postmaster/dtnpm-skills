---
name: dtnpm-systemd-deployment-diagnostics
description: >-
  Guides hardened systemd deployment of `dtnpm` (`dtnpm.service`), Linux file permissions (`0600`/`0700`), `systemd-journal` access for `hdy-stats`, YunoHost/Nginx reverse proxy integration, and runtime diagnostics (`dtnquery`, `hdy-stats`, `journalctl`). Use when deploying `dtnpm` in production or troubleshooting node health, routes, and storage.
---

# Déploiement systemd durci & Diagnostics (`dtnpm`)

Cette compétence rassemble les procédures de déploiement système conformes au Cyber Resilience Act (CRA) et d'exploitation quotidienne de `dtnpm` sous Linux / systemd.

## 1. Déploiement systemd durci (`dtnpm.service`)

### Compte dédié et arborescence sécurisée

Créer un utilisateur système non privilégié et restreindre les droits conformément à `dtnpm/agent_docs/deployment.md` :

```bash
sudo useradd --system --home-dir /var/lib/dtnpm --shell /usr/sbin/nologin dtnpm
sudo install -d -o dtnpm -g dtnpm -m 0700 /var/lib/dtnpm
sudo install -d -o root -g dtnpm -m 0750 /etc/dtnpm
sudo install -d -o dtnpm -g dtnpm -m 0750 /run/dtnpm
```

Permissions impératives des fichiers sensibles :
- `/etc/dtnpm/config.toml` : `0640` (ou `0600` propriétaire `dtnpm:dtnpm` si la configuration porte des secrets).
- `/etc/dtnpm/*.secret`, `keystore.toml`, clés `.key` : **`0600`**.

### Directives de durcissement dans `dtnpm.service`

L'unité fournie dans `dtnpm/dtnpm.service` applique les protections suivantes :
- `User=dtnpm`, `Group=dtnpm`
- `StateDirectory=dtnpm`, `ConfigurationDirectory=dtnpm`
- `ProtectSystem=strict`, `ProtectHome=true`, `NoNewPrivileges=true`
- `CAP_NET_BIND_SERVICE` uniquement si un port < 1024 est requis (sinon privilégier un reverse proxy sur `127.0.0.1:8080`).

Commandes de cycle de vie :

```bash
sudo cp dtnpm.service /etc/systemd/system/dtnpm.service
sudo systemctl daemon-reload
sudo systemctl enable --now dtnpm
sudo systemctl status dtnpm
```

## 2. Intégration Reverse Proxy (YunoHost / Nginx)

Lorsque `[web].address = "127.0.0.1:8080"` est utilisé derrière Nginx/SSO :
1. Configurer `[auth] mode = "proxy"` et `trusted-proxies = ["127.0.0.1"]` dans `/etc/dtnpm/config.toml` (voir `examples/config-iam-yunohost.toml`).
2. Transférer les en-têtes `Remote-User`, `Remote-Email` et `Remote-Groups` uniquement depuis le reverse proxy local (jamais directement depuis des clients externes).
3. Vérifier que le backend `dtnpm` écoute strictement sur `127.0.0.1:8080` pour prévenir tout contournement d'en-têtes d'identité.

## 3. Intégration `hdy-stats` et lecture des journaux

Pour permettre à `hdy-stats` d'interroger le journal systemd sans privilèges `root` complets :
- Ajouter le compte de service exécutant `hdy-stats` au groupe système `systemd-journal` :
  ```bash
  sudo usermod -aG systemd-journal dtnpm
  ```
- Lancer la collecte et consulter le bilan :
  ```bash
  hdy-stats -d /var/lib/dtnpm/hdy-stats.db -r 60 -u dtnpm --show
  ```

Note : le flag `-u/--journald-unit` applique une validation stricte (rejet des tirets initiaux et métacaractères d'injection).

## 4. Check-list de diagnostic en production

En cas d'anomalie (bundle bloqué, CLA déconnecté, quota plein) :

1. **État du démon et logs** :
   ```bash
   sudo journalctl -u dtnpm -n 100 --no-pager
   ```
2. **Identité, endpoints et pairs actifs** :
   ```bash
   dtnquery -c /etc/dtnpm/config.toml info
   dtnquery -c /etc/dtnpm/config.toml eids
   dtnquery -c /etc/dtnpm/config.toml peers
   ```
3. **File d'attente et stockage BPA** :
   ```bash
   dtnquery -c /etc/dtnpm/config.toml bundles -v --limit 25
   dtnquery -c /etc/dtnpm/config.toml store --limit 25
   ```
4. **Permissions socket UNIX & jeton** :
   - Vérifier que `/run/dtnpm/grpc.sock` appartient au groupe configuré (`socket-mode = "0660"`).
   - Vérifier que les utilitaires passent bien `--socket` et `--token-file` lisibles par l'utilisateur courant.
5. **Test de connectivité DTN bout-en-bout** :
   ```bash
   dtnping dtn://<station-distante>/echo -c 3 -w 10s
   ```
