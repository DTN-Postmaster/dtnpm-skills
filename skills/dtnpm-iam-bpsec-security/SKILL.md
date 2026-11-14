---
name: dtnpm-iam-bpsec-security
description: >-
  Configures identity, access control (DTN-HDY-HAM-IAM), gRPC authentication over Unix Domain Sockets, WebUI proxy/local Argon2id auth, Bearer API tokens, and BPSec RFC 9172 HMAC-SHA256 signing/verification (`keystore.toml`, `VerifyPolicy`). Use when securing `dtnpm`, onboarding operators, rotating tokens, or enabling BPSec bundle authentication.
---

# Sécurité IAM (`DTN-HDY-HAM-IAM`) & Authentification BPSec

Cette compétence détaille la mise en place du modèle de sécurité à deux niveaux de `dtnpm` :
1. **IAM local et passerelle (`DTN-HDY-HAM-IAM`)** : authentification des opérateurs WebUI (Argon2id ou SSO reverse proxy), contrôle PBAC par rôle (`Admin`, `Operator`) et protection de l'interface gRPC locale.
2. **BPSec (RFC 9172 — Integrity / BIB HMAC-SHA256)** : authentification des bundles BPv7 de bout en bout tout en restant conforme aux règles radioamateurs (pas de chiffrement de charge utile).

## 1. Comptes WebUI locaux (`dtnpm user`)

Gérer les utilisateurs Argon2id sans exposer les mots de passe dans l'historique shell ou `ps` :

```bash
# Ajouter un administrateur ou opérateur en lisant le mot de passe depuis stdin
printf '%s\n' "$OPERATOR_PASS" | dtnpm user add f4test --role Admin --password-stdin

# Changer le mot de passe d'un utilisateur existant
printf '%s\n' "$NEW_PASS" | dtnpm user passwd f4test --password-stdin

# Lister les comptes (option --data-dir si répertoire hors défaut)
dtnpm user list
```

Rôles reconnus : `Admin` et `Operator`. Éviter l'argument `--password` en ligne de commande (déprécié).

## 2. Jetons API & protection gRPC / Unix Domain Socket (`[grpc.auth]`)

### Émettre et révoquer des jetons persistants (`dtnpm token`)

```bash
# Crée un jeton et affiche le secret brut une seule fois (le sauvegarder en mode 0600)
dtnpm token add cli-ops --role Operator > /etc/dtnpm/cli-token.secret
chmod 0600 /etc/dtnpm/cli-token.secret

dtnpm token list
dtnpm token revoke cli-ops
```

### Configuration `[grpc.auth]` et `[auth]` dans `config.toml`

Référence issue de `examples/config-iam-yunohost.toml` :

```toml
[grpc.auth]
enabled = true
socket-path = "/run/dtnpm/grpc.sock"
socket-group = "dtnpm"
socket-mode = "0660"

# Jeton dédié (ex. pour un CLA ou service)
[[grpc.auth.tokens]]
name = "cla-hqfbp"
role = "Operator"
# Utiliser le hash stocké ou la gestion par `dtnpm token add` selon votre déploiement

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

Toujours connecter les clients CLI et CLAs externes via `--socket /run/dtnpm/grpc.sock --token-file /etc/dtnpm/cli-token.secret`.

## 3. Signature et vérification BPSec (RFC 9172 / HMAC-SHA256)

### Créer un fichier de clé partagée sécurisé (`0600`)

```bash
umask 0077
openssl rand -hex 32 > ~/.config/dtn/f4test.key
```

### Configurer `~/.config/dtn/keystore.toml`

Format vérifié dans `dtnpm-core/src/cli_security.rs` (`KeystoreConfig` / `KeyEntryConfig`) :

```toml
[[keys]]
eid = "dtn://f4test/*"
alg = "hmac-sha256"
key_file = "/home/operateur/.config/dtn/f4test.key"

[[keys]]
eid = "dtn://f4peer/*"
alg = "hmac-sha256"
key_file = "/home/operateur/.config/dtn/f4peer.key"
```

Sécuriser le trousseau : `chmod 0600 ~/.config/dtn/keystore.toml`.

### Politiques de vérification (`--verify-policy`)

- **`strict`** : rejette tout bundle non signé (`Unsigned`), à clé inconnue (`MissingKey`) ou invalide (`Invalid`). Recommandé pour les services exposés (`dtnbasket`, `dtntrigger`, `template-dtnpm-app`).
- **`warn`** : journalise un avertissement sur `stderr` si la signature manque ou échoue, mais délivre la charge utile (défaut de `dtnprint` pour l'interopérabilité progressive).
- **`ignore`** : désactive la vérification (banc de test uniquement).

Exemple signature + vérification bout-en-bout :

```bash
# Réception stricte
dtnprint -s incoming --verify-policy strict --keystore ~/.config/dtn/keystore.toml

# Envoi signé
dtnsend -r dtn://f4peer/incoming --sign-key-file ~/.config/dtn/f4test.key --security-source dtn://f4test/ payload.txt
```
