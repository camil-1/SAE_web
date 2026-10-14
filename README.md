# SAÉ : Infrastructure Nextcloud + Keycloak + OpenLDAP + Lstu + OAuth2 Proxy

Mise en place d'une infrastructure liant un Nextcloud, un Keycloak, un
serveur OpenLDAP, un serveur Lstu et un OAuth2 Proxy, avec Matomo en
bonus, sur une unique VM Debian 13, suivant les instructions officielles
de chaque logiciel.

Notre VM finale configurée est téléchargeable ici : <https://nextcloud.doucey.fr/s/kkPYfQKaK8d7YCn>

## Documents

- [`docs/tutoriel.md`](docs/tutoriel.md) : tutoriel détaillé pour monter
  l'infrastructure de bout en bout, y compris la récupération de l'image
  de VM finale dans virt-manager.
- [`docs/methodologie.md`](docs/methodologie.md) : méthodes de travail du
  groupe (répartition des tâches, planning, outils).
- [`docs/autocritique.md`](docs/autocritique.md) : les difficultés
  rencontrées pendant le montage et ce qui serait fait différemment.

## Contenu du dépôt

- `sae.pdf` : sujet de la SAÉ.
- `pki/` : autorité de certification créée pour l'infrastructure,
  certificats et **clés privées** inclus (voir §4 du sujet et
  `docs/tutoriel.md`, Phase 0).
- `secrets/` : mots de passe et secrets générés pendant le montage
  (admin LDAP, Keycloak, bases de données, clients OAuth2...).
- `docs/` : les trois documents rédactionnels demandés par le sujet.
