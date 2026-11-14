# Politique de sécurité — `skills`

## Bonnes pratiques documentaires

Les skills de ce dépôt décrivent des configurations système, réseau, systemd et de sécurité cryptographique (IAM + BPSec). Toute documentation erronée peut amener un opérateur à exposer son nœud DTN sans authentification ou à fuiter des secrets dans l'historique shell ou `/proc`.

Toute contribution doit donc :
- Privilégier `--token-file`, `DTN_HDY_TOKEN_FILE`, `--password-stdin` et les Unix Domain Sockets authentifiés.
- Signaler clairement les implications des drapeaux d'opt-in dangereux (`--allow-unauthenticated-udp`, `--allow-unidentified-rf`, `--verify-policy ignore`, `--allow-third-party-report-to`, `allow_private_ips`).
- Rappel permanent radioamateur : le chiffrement de charge utile est interdit sur les fréquences amateurs ; seule **l'authentification / intégrité (BPSec BIB HMAC-SHA256)** et le trafic en clair sont conformes en RF amateur.

## Signaler un problème de sécurité

Contacter `security@dtn-postmaster.org` ou ouvrir une *Security Advisory* privée sur GitHub pour toute recommandation dangereuse ou vulnérabilité détectée.
