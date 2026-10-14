# Tutoriel : Infrastructure SAÉ (Nextcloud + Keycloak + OpenLDAP + Lstu + OAuth2 Proxy)

Ce tutoriel décrit pas à pas la mise en place de l'infrastructure sur une unique
machine Debian 13 (trixie), en suivant les instructions officielles de chaque
logiciel (hors Docker), avec Nginx comme reverse-proxy TLS en frontal.

## Préambule : récupérer et faire tourner la VM fournie dans virt-manager

L'infrastructure décrite dans ce tutoriel est fournie telle quelle sous
forme d'image disque de VM déjà entièrement configurée (toutes les
phases ci-dessous appliquées), afin que n'importe qui puisse la faire
tourner sans tout réinstaller. Cette partie explique comment on exporte
cette image depuis la machine où elle a été montée, et comment l'importer
ailleurs dans virt-manager.

Caractéristiques de la VM : Debian 13 (trixie), 8192 Mo de RAM, 8 vCPU,
disque de 20 Go (qcow2), chipset **Q35**, carte réseau sur le réseau
virtuel par défaut de libvirt (**NAT**, `virbr0`, `192.168.122.0/24`).

### P.1 Export de l'image (côté machine source)

Éteindre proprement la VM avant de copier son disque (une image copiée à
chaud serait incohérente) :

```bash
sudo virsh shutdown <nom-de-la-vm>
```

Récupérer le chemin du fichier disque associé à la VM :

```bash
sudo virsh domblklist <nom-de-la-vm>
```

Celui-ci se trouve en général sous `/var/lib/libvirt/images/`. Une fois
la VM arrêtée, copier ce fichier `.qcow2` (clé USB, partage réseau...) ;
sa taille peut être réduite avant transfert avec :

```bash
sudo qemu-img convert -O qcow2 -c /var/lib/libvirt/images/<nom-de-la-vm>.qcow2 vm-sae-finale.qcow2
```

### P.2 Import dans virt-manager (côté machine destination)

Copier le fichier `.qcow2` obtenu à l'étape précédente sur la machine de
destination (par exemple dans `/var/lib/libvirt/images/`), puis dans
virt-manager : **Fichier → Nouvelle machine virtuelle → Importer une
image disque existante**.

1. Parcourir jusqu'au fichier `.qcow2` copié.
2. Système d'exploitation : Debian 13 (ou "generic Linux" si non détecté).
3. Mémoire et CPU : **8192 Mo de RAM**, **8 vCPU** (à réduire si la
   machine de destination dispose de moins de ressources : l'infra
   démarre avec moins, au prix de performances dégradées, dans une
   mesure qui n'a pas été testée précisément).
4. Réseau : réseau virtuel par défaut (**NAT**, `virbr0`). S'il n'existe
   pas encore sur la machine de destination, le démarrer avant de
   lancer la VM :
   ```bash
   sudo virsh net-start default
   ```
5. Avant de terminer, cocher **personnaliser la configuration avant
   l'installation**, pour vérifier que le chipset machine (onglet *Vue
   d'ensemble*) est bien **Q35** (virt-manager peut proposer i440FX par
   défaut selon la détection du système).
6. Terminer puis démarrer la VM.

### P.3 Vérification de l'adresse IP

La configuration réseau à l'intérieur de la VM est **statique**
(`/etc/network/interfaces`, adresse `192.168.122.10/24`, passerelle
`192.168.122.1`) plutôt qu'en DHCP, précisément pour que cette adresse
reste la même après un export/import, quelle que soit la machine hôte,
à condition que celle-ci utilise elle aussi le réseau NAT par défaut de
libvirt en `192.168.122.0/24` (le cas le plus courant, inchangé par
défaut sur une installation standard de virt-manager).

Vérifier que la VM répond bien à cette adresse :

```bash
ping -c2 192.168.122.10
ssh <utilisateur>@192.168.122.10
```

Si la machine de destination utilise un sous-réseau NAT différent,
reconfigurer l'adresse statique dans la VM (`/etc/network/interfaces`)
en conséquence, et adapter toutes les adresses utilisées dans le reste
de ce tutoriel.

> Testé : au tout premier démarrage après un import (nouvelle adresse
> MAC générée pour la VM), le réseau peut ne pas répondre immédiatement
> malgré la configuration statique. En cause, `networking.service` peut
> démarrer avant que l'interface virtio ait fini d'être nommée par udev.
> Un simple redémarrage de la VM suffit à résoudre ce point ; les
> démarrages suivants n'ont pas ce problème.

À partir d'ici, le reste de ce tutoriel s'exécute **sur cette VM** (sauf
mention contraire, pour les quelques commandes à exécuter sur le poste
client).

## Prérequis

- Une machine Debian GNU/Linux 13 (trixie), avec un utilisateur disposant des
  droits `sudo`.
- Un nom d'hôte accessible depuis le poste client (ici `192.168.122.10`).
- Un poste client sur lequel on ajoutera les entrées DNS locales et la CA de
  confiance pour tester l'infrastructure dans un navigateur.

## Convention de nommage

Tous les services sont exposés sous le domaine `sae.local`, résolu localement
via `/etc/hosts` (pas de vrai DNS) :

| Service       | Nom d'hôte               |
|---------------|--------------------------|
| Nextcloud     | `nextcloud.sae.local`    |
| Keycloak      | `keycloak.sae.local`     |
| Lstu          | `lstu.sae.local`         |
| Matomo (bonus)| `matomo.sae.local`       |

> OAuth2 Proxy n'a pas de sous-domaine propre : il est exposé sous
> `lstu.sae.local/oauth2/*` (voir Phase 5), puisqu'il protège directement
> les routes de ce site plutôt que de servir de portail séparé.

## Phase 0 : Socle (DNS local, PKI, reverse-proxy)

### 0.1 Résolution de noms

Sur le **serveur** ET sur le **poste client**, ajouter dans `/etc/hosts` (en
remplaçant l'IP par celle de votre serveur) :

```
192.168.122.10 nextcloud.sae.local keycloak.sae.local lstu.sae.local matomo.sae.local
```

### 0.2 Autorité de certification et certificat wildcard

Nextcloud (et les navigateurs modernes en général) exigent une connexion
chiffrée valide. On crée une autorité de certification locale, puis un
certificat **wildcard** `*.sae.local` couvrant tous les services (plus simple
à gérer qu'un certificat par service, au prix d'un cert un peu moins
cloisonné, ce qui reste acceptable pour un TP).

Sur le serveur :

```bash
mkdir -p ~/pki && cd ~/pki

# Autorité de certification (valable 5 ans)
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 1825 -key ca.key -out ca.crt \
  -subj "/C=FR/ST=Lorraine/L=Nancy/O=SAE-DACS/CN=SAE-DACS-CA"

# Clé + CSR pour le certificat wildcard
openssl genrsa -out wildcard.sae.local.key 4096
openssl req -new -key wildcard.sae.local.key -out wildcard.sae.local.csr \
  -subj "/C=FR/ST=Lorraine/L=Nancy/O=SAE-DACS/CN=*.sae.local"

# Extension SAN (obligatoire : les navigateurs ignorent le CN seul depuis 2017)
cat > wildcard.ext <<'EOF'
subjectAltName = DNS:*.sae.local, DNS:sae.local
EOF

# Signature du certificat par notre CA (valable ~825 jours, max accepté)
openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial \
  -days 825 -in wildcard.sae.local.csr -out wildcard.sae.local.crt \
  -extfile wildcard.ext
```

Conservez précieusement `ca.key`, `ca.crt`, `wildcard.sae.local.key` et
`wildcard.sae.local.crt`.

Installer la CA dans le magasin de confiance du serveur :

```bash
sudo cp ~/pki/ca.crt /usr/local/share/ca-certificates/sae-dacs-ca.crt
sudo update-ca-certificates
```

Installer la CA sur le poste client (pour que le navigateur fasse
confiance aux services sans avertissement) :

- Récupérer `ca.crt` (ex: `scp serveur:~/pki/ca.crt .`)
- Sur Debian/Ubuntu client : mêmes commandes que ci-dessus.
- Sur Arch Linux :
  ```bash
  sudo cp ca.crt /etc/ca-certificates/trust-source/anchors/
  sudo trust extract-compat
  ```
- Le cas échéant, importer aussi la CA dans le magasin de certificats de
  votre navigateur (Firefox et Chrome gèrent parfois leur propre magasin
  séparé du magasin système).

Déployer le certificat wildcard pour Nginx :

```bash
sudo mkdir -p /etc/nginx/ssl
sudo cp ~/pki/wildcard.sae.local.crt /etc/nginx/ssl/
sudo cp ~/pki/wildcard.sae.local.key /etc/nginx/ssl/
sudo chmod 600 /etc/nginx/ssl/wildcard.sae.local.key
sudo chmod 644 /etc/nginx/ssl/wildcard.sae.local.crt
```

### 0.3 Reverse-proxy Nginx

Installation depuis les paquets Debian (autorisé par le sujet) :

```bash
sudo apt update
sudo apt install -y nginx
```

Chaque service aura son propre vhost dans `/etc/nginx/sites-available/`,
référençant le certificat wildcard commun :

```nginx
server {
    listen 443 ssl;
    server_name <nom-du-service>.sae.local;

    ssl_certificate     /etc/nginx/ssl/wildcard.sae.local.crt;
    ssl_certificate_key /etc/nginx/ssl/wildcard.sae.local.key;

    location / {
        proxy_pass http://127.0.0.1:<port-interne-du-service>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

Les vhosts spécifiques à chaque service sont détaillés dans les phases
suivantes.

Vérification (test réalisé avec un vhost temporaire) :

```bash
curl https://nextcloud.sae.local/
```

Doit répondre `200 OK` sans erreur de certificat (preuve que la CA est bien
approuvée côté client et que le certificat wildcard est correctement servi).

---

## Phase 1 : OpenLDAP

Installation depuis les paquets Debian (`slapd`), configuration automatisée
via `debconf-set-selections` (méthode documentée par Debian pour éviter les
prompts interactifs).

### 1.1 Préseeding et installation

Générer un mot de passe administrateur LDAP fort, puis préparer les réponses
debconf (remplacer `<MOT_DE_PASSE>`) :

```bash
cat > slapd-preseed.txt <<EOF
slapd slapd/internal/generated_adminpw password <MOT_DE_PASSE>
slapd slapd/internal/adminpw password <MOT_DE_PASSE>
slapd slapd/password1 password <MOT_DE_PASSE>
slapd slapd/password2 password <MOT_DE_PASSE>
slapd slapd/domain string sae.local
slapd shared/organization string SAE
slapd slapd/backend select MDB
slapd slapd/purge_database boolean true
slapd slapd/move_old_database boolean true
slapd slapd/allow_ldap_v2 boolean false
slapd slapd/no_configuration boolean false
EOF

sudo debconf-set-selections slapd-preseed.txt
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y slapd ldap-utils
rm slapd-preseed.txt
```

Le domaine `sae.local` donne automatiquement la base `dc=sae,dc=local`. Les
schémas `core`, `cosine`, `nis` et `inetorgperson` sont chargés par défaut par
le paquet Debian.

### 1.2 Vérification

```bash
ldapwhoami -x -D "cn=admin,dc=sae,dc=local" -w <MOT_DE_PASSE>
```

### 1.3 Arborescence et utilisateurs de test

```
dc=sae,dc=local
├── ou=people
│   ├── uid=alice
│   └── uid=bob
└── ou=groups
    ├── cn=nextcloud-users (member: alice, bob)
    └── cn=lstu-admins     (member: bob)
```

LDIF (`base-structure.ldif`) :

```ldif
dn: ou=people,dc=sae,dc=local
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=sae,dc=local
objectClass: organizationalUnit
ou: groups

dn: uid=alice,ou=people,dc=sae,dc=local
objectClass: inetOrgPerson
uid: alice
cn: Alice Dupont
sn: Dupont
givenName: Alice
mail: alice@sae.local
userPassword: <hash SSHA généré via slappasswd -s>

dn: uid=bob,ou=people,dc=sae,dc=local
objectClass: inetOrgPerson
uid: bob
cn: Bob Martin
sn: Martin
givenName: Bob
mail: bob@sae.local
userPassword: <hash SSHA généré via slappasswd -s>

dn: cn=nextcloud-users,ou=groups,dc=sae,dc=local
objectClass: groupOfNames
cn: nextcloud-users
member: uid=alice,ou=people,dc=sae,dc=local
member: uid=bob,ou=people,dc=sae,dc=local

dn: cn=lstu-admins,ou=groups,dc=sae,dc=local
objectClass: groupOfNames
cn: lstu-admins
member: uid=bob,ou=people,dc=sae,dc=local
```

Le hash SSHA d'un mot de passe se génère avec :

```bash
slappasswd -s 'MotDePasseUtilisateur'
```

Chargement :

```bash
ldapadd -x -H ldap://localhost -D "cn=admin,dc=sae,dc=local" -w <MOT_DE_PASSE> -f base-structure.ldif
```

Vérification (bind en tant qu'utilisateur normal) :

```bash
ldapwhoami -x -D "uid=alice,ou=people,dc=sae,dc=local" -w 'MotDePasseUtilisateur'
```

> Les objectClass `posixAccount`/`posixGroup` n'ont volontairement pas été
> utilisées : seuls `inetOrgPerson` et `groupOfNames` sont nécessaires pour
> l'authentification web (Keycloak s'appuie par défaut sur `inetOrgPerson`),
> aucun compte Unix local n'est requis.

### 1.4 Restriction de l'écoute réseau

Par défaut `slapd` écoute sur toutes les interfaces (`0.0.0.0:389`). Seul
Keycloak (colocalisé sur la même machine) interroge OpenLDAP : on restreint
donc l'écoute à la boucle locale, comme le recommande la page de manuel
`slapd(8)` :

```bash
sudo sed -i 's|^SLAPD_SERVICES=.*|SLAPD_SERVICES="ldap://127.0.0.1/ ldapi:///"|' /etc/default/slapd
sudo systemctl restart slapd
```

## Phase 2 : Keycloak

Installation officielle (archive zip, hors Docker), en mode production
(`kc.sh start --optimized`) derrière Nginx, base de données MariaDB, réalisée
entièrement via la CLI d'administration officielle `kcadm.sh` pour rester
scriptable et reproductible.

### 2.1 Prérequis : Java et base de données

Keycloak 26.x nécessite JDK 17 ou 21. MariaDB est réutilisée plus tard pour
Nextcloud.

```bash
sudo apt install -y openjdk-21-jdk-headless mariadb-server
```

```sql
CREATE DATABASE keycloak CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'keycloak'@'localhost' IDENTIFIED BY '<MOT_DE_PASSE_DB>';
GRANT ALL PRIVILEGES ON keycloak.* TO 'keycloak'@'localhost';
FLUSH PRIVILEGES;
```

### 2.2 Installation de Keycloak

Téléchargement de la distribution officielle (zip) depuis les releases
GitHub de Keycloak (même contenu que sur keycloak.org/downloads) :

```bash
sudo useradd -r -m -d /opt/keycloak -s /usr/sbin/nologin keycloak
curl -sLO https://github.com/keycloak/keycloak/releases/download/26.7.4/keycloak-26.7.4.zip
unzip -q keycloak-26.7.4.zip
sudo rsync -a keycloak-26.7.4/ /opt/keycloak/
sudo chown -R keycloak:keycloak /opt/keycloak
```

### 2.3 Configuration (mode production, reverse-proxy)

`/opt/keycloak/conf/keycloak.conf` :

```properties
db=mariadb
db-url=jdbc:mariadb://localhost:3306/keycloak
db-username=keycloak
db-password=<MOT_DE_PASSE_DB>

hostname=keycloak.sae.local
http-enabled=true
http-host=127.0.0.1
http-port=8080
proxy-headers=xforwarded
cache=local
```

> `cache=local` désactive le cache distribué Infinispan/JGroups (et le port
> réseau de cluster associé, ouvert par défaut) : ce réglage est la
> configuration officiellement documentée pour un nœud Keycloak unique,
> non-HA, ce qui est notre cas ici (une seule instance, un seul serveur).

Compilation de la distribution optimisée (obligatoire avant le premier
démarrage en mode production) :

```bash
sudo -u keycloak /opt/keycloak/bin/kc.sh build
```

### 2.4 Service systemd

`/etc/systemd/system/keycloak.service` :

```ini
[Unit]
Description=Keycloak
After=network.target mariadb.service

[Service]
Type=idle
User=keycloak
Group=keycloak
Environment=KC_BOOTSTRAP_ADMIN_USERNAME=admin
Environment=KC_BOOTSTRAP_ADMIN_PASSWORD=<MOT_DE_PASSE_ADMIN>
ExecStart=/opt/keycloak/bin/kc.sh start --optimized
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Les variables `KC_BOOTSTRAP_ADMIN_*` ne servent qu'à la toute première
initialisation (création du compte admin) ; elles sont ignorées ensuite.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now keycloak
```

### 2.5 Vhost Nginx

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name keycloak.sae.local;

    ssl_certificate     /etc/nginx/ssl/wildcard.sae.local.crt;
    ssl_certificate_key /etc/nginx/ssl/wildcard.sae.local.key;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port 443;
    }
}
```

Vérification : `https://keycloak.sae.local/realms/master/.well-known/openid-configuration`
doit répondre avec `"issuer":"https://keycloak.sae.local/realms/master"`.

### 2.6 Création du realm et connexion à la CLI

```bash
/opt/keycloak/bin/kcadm.sh config credentials \
  --server http://127.0.0.1:8080 --realm master \
  --user admin --password <MOT_DE_PASSE_ADMIN> --client admin-cli

/opt/keycloak/bin/kcadm.sh create realms \
  -s realm=sae -s enabled=true -s sslRequired=external -s displayName=SAE
```

> `kcadm.sh` est utilisé en ciblant directement le backend `http://127.0.0.1:8080`
> (loopback) plutôt que le vhost HTTPS, pour éviter d'avoir à importer la CA
> maison dans le magasin de confiance Java. Cela n'affecte pas les URLs
> générées pour les utilisateurs finaux (contrôlées par `hostname=` dans
> `keycloak.conf`).

### 2.7 Fédération LDAP (OpenLDAP comme source d'authentification)

Composant `ldap` créé via l'API components, en **lecture seule**
(`editMode=READ_ONLY`) puisque OpenLDAP reste l'unique source de vérité :

```json
{
  "name": "openldap",
  "providerId": "ldap",
  "providerType": "org.keycloak.storage.UserStorageProvider",
  "parentId": "<ID_DU_REALM_SAE>",
  "config": {
    "enabled": ["true"],
    "vendor": ["other"],
    "usernameLDAPAttribute": ["uid"],
    "rdnLDAPAttribute": ["uid"],
    "uuidLDAPAttribute": ["entryUUID"],
    "userObjectClasses": ["inetOrgPerson, organizationalPerson"],
    "connectionUrl": ["ldap://localhost:389"],
    "usersDn": ["ou=people,dc=sae,dc=local"],
    "authType": ["simple"],
    "bindDn": ["cn=admin,dc=sae,dc=local"],
    "bindCredential": ["<MOT_DE_PASSE_LDAP>"],
    "searchScope": ["1"],
    "editMode": ["READ_ONLY"],
    "syncRegistrations": ["false"],
    "pagination": ["true"],
    "importEnabled": ["true"],
    "trustEmail": ["true"],
    "batchSizeForSync": ["1000"],
    "fullSyncPeriod": ["-1"],
    "changedSyncPeriod": ["-1"]
  }
}
```

```bash
/opt/keycloak/bin/kcadm.sh create components -r sae -f ldap-federation.json
```

Synchronisation complète (récupère les utilisateurs LDAP dans Keycloak) :

```bash
TOKEN=$(curl -s -X POST http://127.0.0.1:8080/realms/master/protocol/openid-connect/token \
  -d grant_type=password -d client_id=admin-cli -d username=admin -d password=<MOT_DE_PASSE_ADMIN> \
  | grep -oP '(?<="access_token":")[^"]+')

curl -s -X POST "http://127.0.0.1:8080/admin/realms/sae/user-storage/<ID_COMPOSANT>/sync?action=triggerFullSync" \
  -H "Authorization: Bearer $TOKEN"
```

Vérification : `alice` et `bob` apparaissent dans `kcadm.sh get users -r sae`,
et un utilisateur peut obtenir un token avec son mot de passe LDAP :

```bash
curl -s -X POST http://127.0.0.1:8080/realms/sae/protocol/openid-connect/token \
  -d grant_type=password -d client_id=admin-cli -d username=alice -d password=ChangeMe123!
```

### 2.7bis Mapper de groupes LDAP (nécessaire pour les restrictions par groupe)

Problème rencontré en Phase 5 : la fédération LDAP ci-dessus importe les
utilisateurs, mais pas leurs appartenances aux groupes LDAP. Keycloak
ignore totalement les groupes `nextcloud-users`/`lstu-admins` tant qu'aucun
mapper dédié n'est ajouté. Résultat concret : une restriction par groupe
côté OAuth2 Proxy (Phase 5) rejette tout le monde, y compris les membres du
bon groupe, car leur token OIDC ne contient aucun claim `groups`.

Correction : ajouter un sous-mapper `group-ldap-mapper` à la fédération :

```json
{
  "name": "group-mapper",
  "providerId": "group-ldap-mapper",
  "providerType": "org.keycloak.storage.ldap.mappers.LDAPStorageMapper",
  "parentId": "<ID_DU_COMPOSANT_LDAP>",
  "config": {
    "groups.dn": ["ou=groups,dc=sae,dc=local"],
    "group.name.ldap.attribute": ["cn"],
    "group.object.classes": ["groupOfNames"],
    "membership.ldap.attribute": ["member"],
    "membership.attribute.type": ["DN"],
    "membership.user.ldap.attribute": ["uid"],
    "mode": ["READ_ONLY"],
    "user.roles.retrieve.strategy": ["LOAD_GROUPS_BY_MEMBER_ATTRIBUTE"],
    "preserve.group.inheritance": ["false"],
    "ignore.missing.groups": ["false"],
    "drop.non.existing.groups.during.sync": ["false"]
  }
}
```

```bash
/opt/keycloak/bin/kcadm.sh create components -r sae -f ldap-group-mapper.json
```

Puis synchronisation (import des groupes + appartenances) :

```bash
TOKEN=$(curl -s -X POST http://127.0.0.1:8080/realms/master/protocol/openid-connect/token \
  -d grant_type=password -d client_id=admin-cli -d username=admin -d password=<MOT_DE_PASSE_ADMIN> \
  | grep -oP '(?<="access_token":")[^"]+')

curl -s -X POST "http://127.0.0.1:8080/admin/realms/sae/user-storage/<ID_COMPOSANT_LDAP>/mappers/<ID_MAPPER_GROUPE>/sync?direction=fedToKeycloak" \
  -H "Authorization: Bearer $TOKEN"
```

Vérification : `kcadm.sh get users/<ID_UTILISATEUR>/groups -r sae` doit
lister `/lstu-admins` et/ou `/nextcloud-users` selon l'utilisateur.

### 2.8 Clients OIDC

Client Nextcloud (confidentiel, Authorization Code flow) :

```bash
/opt/keycloak/bin/kcadm.sh create clients -r sae \
  -s clientId=nextcloud \
  -s protocol=openid-connect \
  -s publicClient=false \
  -s standardFlowEnabled=true \
  -s directAccessGrantsEnabled=false \
  -s 'redirectUris=["https://nextcloud.sae.local/*"]' \
  -s 'webOrigins=["https://nextcloud.sae.local"]'
```

Client OAuth2 Proxy (confidentiel, Authorization Code flow) :

```bash
/opt/keycloak/bin/kcadm.sh create clients -r sae \
  -s clientId=oauth2-proxy \
  -s protocol=openid-connect \
  -s publicClient=false \
  -s standardFlowEnabled=true \
  -s directAccessGrantsEnabled=false \
  -s 'redirectUris=["https://lstu.sae.local/oauth2/callback"]' \
  -s 'webOrigins=["https://lstu.sae.local"]'
```

Récupération du secret de chaque client (nécessaire pour Nextcloud et
OAuth2 Proxy) :

```bash
/opt/keycloak/bin/kcadm.sh get clients/<ID_CLIENT>/client-secret -r sae
```

## Phase 3 : Nextcloud

Installation officielle (archive `.tar.bz2` téléchargée depuis nextcloud.com,
hors Docker/Snap), PHP-FPM + MariaDB, Nginx avec la configuration exemple
officielle, connexion à Keycloak via l'app officielle `user_oidc`.

### 3.1 PHP et extensions

Debian 13 fournit PHP 8.4, supporté par Nextcloud. Paquets installés : la
liste **requise** plus les modules **recommandés généraux** de la page
officielle *Preparing PHP* (`intl`, `sodium`, `sysvsem`, `APCu`) :

```bash
sudo apt install -y \
  php8.4-fpm php8.4-common php8.4-curl php8.4-gd php8.4-mbstring \
  php8.4-xml php8.4-zip php8.4-mysql php8.4-intl php8.4-apcu php8.4-opcache
```

> `ctype`, `fileinfo`, `posix`, `session`, `zlib`, `openssl`, `sysvsem` et
> `sodium` sont déjà fournis par `php8.4-common`/le cœur de PHP 8.4 sous
> Debian (vérifié avec `php8.4 -m`). Aucun module supplémentaire
> (`ldap`, `smbclient`, `imagick`, `bcmath`, `gmp`…) n'a été installé : ils ne
> sont listés que comme *optionnels pour des usages spécifiques* que ce TP
> n'utilise pas.

### 3.2 Base de données

```sql
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY '<MOT_DE_PASSE_DB>';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost';
FLUSH PRIVILEGES;
```

### 3.3 Téléchargement et déploiement

```bash
curl -sLO https://download.nextcloud.com/server/releases/latest.tar.bz2
curl -sLO https://download.nextcloud.com/server/releases/latest.tar.bz2.sha256
sha256sum -c latest.tar.bz2.sha256

tar -xjf latest.tar.bz2
sudo rsync -a nextcloud/ /var/www/nextcloud/
sudo chown -R www-data:www-data /var/www/nextcloud/
```

### 3.4 PHP-FPM

Le pool par défaut (`/etc/php/8.4/fpm/pool.d/www.conf`, socket Unix
`/run/php/php8.4-fpm.sock`, utilisateur `www-data`) est réutilisé tel quel.
Seuls les réglages explicitement documentés comme nécessaires sont ajoutés
(minimum officiel `memory_limit` ≥ 512 Mo ; `upload_max_filesize`/
`post_max_size` alignés sur la limite Nginx de 512 Mo) :

```ini
php_admin_value[memory_limit] = 512M
php_admin_value[upload_max_filesize] = 512M
php_admin_value[post_max_size] = 512M
```

```bash
sudo systemctl restart php8.4-fpm
```

### 3.5 Vhost Nginx

Configuration reprise **telle quelle** depuis l'exemple officiel
(`admin_manual/installation/nginx-root.conf.sample`), seules trois choses
adaptées : le nom de domaine, le chemin du certificat (wildcard commun) et
le backend PHP-FPM (socket Unix du pool par défaut au lieu de TCP
`127.0.0.1:9000`) :

```nginx
upstream php-handler {
    server unix:/run/php/php8.4-fpm.sock;
}

map $arg_v $asset_immutable {
    "" "";
    default ", immutable";
}

server {
    listen 80;
    listen [::]:80;
    server_name nextcloud.sae.local;
    server_tokens off;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name nextcloud.sae.local;

    root /var/www/nextcloud;

    ssl_certificate     /etc/nginx/ssl/wildcard.sae.local.crt;
    ssl_certificate_key /etc/nginx/ssl/wildcard.sae.local.key;

    server_tokens off;

    client_max_body_size 512M;
    client_body_timeout 300s;
    fastcgi_buffers 64 4K;

    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml text/javascript application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/wasm application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

    client_body_buffer_size 512k;

    add_header Referrer-Policy                   "no-referrer"       always;
    add_header X-Content-Type-Options            "nosniff"           always;
    add_header X-Frame-Options                   "SAMEORIGIN"        always;
    add_header X-Permitted-Cross-Domain-Policies "none"              always;
    add_header X-Robots-Tag                      "noindex, nofollow" always;

    fastcgi_hide_header X-Powered-By;

    include mime.types;
    types {
        text/javascript mjs;
    }

    index index.php index.html /index.php$request_uri;

    location = / {
        if ( $http_user_agent ~ ^DavClnt ) {
            return 302 /remote.php/webdav/$is_args$args;
        }
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location ^~ /.well-known {
        location = /.well-known/carddav { return 301 /remote.php/dav/; }
        location = /.well-known/caldav  { return 301 /remote.php/dav/; }
        location /.well-known/acme-challenge    { try_files $uri $uri/ =404; }
        location /.well-known/pki-validation    { try_files $uri $uri/ =404; }
        return 301 /index.php$request_uri;
    }

    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)  { return 404; }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console)                { return 404; }
    location ~ ^/(?:composer\.(?:json|lock)|package(?:-lock)?\.json|core/shipped\.json)$ { return 404; }

    location ~ \.php(?:$|/) {
        rewrite ^/(?!index|remote|public|cron|status|ocs\/v[12]|ocs-provider\/.+|core\/ajax\/update|updater\/.+|.+\/richdocumentscode(_arm64)?\/proxy) /index.php$request_uri;

        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;
        try_files $fastcgi_script_name =404;

        include fastcgi_params;
        fastcgi_pass php-handler;

        fastcgi_param SCRIPT_FILENAME            $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO                  $path_info;
        fastcgi_param HTTPS                      on;
        fastcgi_param modHeadersAvailable        true;
        fastcgi_param front_controller_active    true;

        fastcgi_intercept_errors on;
        fastcgi_request_buffering on;
        fastcgi_max_temp_file_size 0;
    }

    location ~ \.(?:css|js|mjs|svg|gif|ico|jpg|png|webp|wasm|tflite|map|ogg|flac|mp4|webm)$ {
        try_files $uri /index.php$request_uri;
        add_header Cache-Control                     "public, max-age=15778463$asset_immutable";
        add_header Referrer-Policy                   "no-referrer"       always;
        add_header X-Content-Type-Options            "nosniff"           always;
        add_header X-Frame-Options                   "SAMEORIGIN"        always;
        add_header X-Permitted-Cross-Domain-Policies "none"              always;
        add_header X-Robots-Tag                      "noindex, nofollow" always;
        access_log off;
    }

    location ~ \.(otf|woff2?)$ {
        try_files $uri /index.php$request_uri;
        expires 7d;
        access_log off;
    }

    location /remote {
        return 301 /remote.php$request_uri;
    }

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }
}
```

> La ligne `application/wasm wasm;` du bloc `types` de l'exemple officiel a
> été retirée : Nginx la déclare déjà par défaut dans `mime.types` sous
> Debian, la garder générait un warning de type dupliqué au chargement.

### 3.6 Installation en ligne de commande

```bash
sudo -u www-data php /var/www/nextcloud/occ maintenance:install \
  --database mysql --database-name nextcloud --database-user nextcloud \
  --database-pass <MOT_DE_PASSE_DB> \
  --admin-user admin --admin-pass <MOT_DE_PASSE_ADMIN>

sudo -u www-data php /var/www/nextcloud/occ config:system:set trusted_domains 1 --value=nextcloud.sae.local
sudo -u www-data php /var/www/nextcloud/occ config:system:set overwriteprotocol --value=https
sudo -u www-data php /var/www/nextcloud/occ config:system:set overwrite.cli.url --value=https://nextcloud.sae.local
```

Vérification : `occ status` (installed: true) et `occ check` (aucune sortie
= aucun avertissement).

### 3.7 Tâches planifiées (cron)

Méthode **Cron** recommandée par la documentation officielle (plutôt
qu'AJAX ou Webcron) :

```bash
sudo crontab -u www-data -e
```

```
*/5 * * * * php -f /var/www/nextcloud/cron.php
```

> Nextcloud bascule automatiquement en mode "Cron" dès que `cron.php` est
> exécuté par une tâche planifiée, sans aucune action nécessaire dans
> l'interface d'administration.

### 3.8 SSO via Keycloak (app officielle `user_oidc`)

```bash
sudo -u www-data php /var/www/nextcloud/occ app:install user_oidc

sudo -u www-data php /var/www/nextcloud/occ user_oidc:provider keycloak \
  --clientid="nextcloud" \
  --clientsecret="<SECRET_CLIENT_NEXTCLOUD>" \
  --discoveryuri="https://keycloak.sae.local/realms/sae/.well-known/openid-configuration"
```

Point bloquant rencontré et résolu : la première tentative échoue avec
`Host keycloak.sae.local was not connected to because it violates local
access rules`, une protection anti-SSRF intégrée à Nextcloud qui bloque
par défaut les requêtes serveur vers des adresses IP privées (RFC1918). Keycloak
étant nécessairement sur le réseau privé du TP, le paramètre officiellement
documenté pour ce cas (*Proxy Configurations*, `allow_local_remote_servers`)
est activé :

```bash
sudo -u www-data php /var/www/nextcloud/occ config:system:set allow_local_remote_servers --value=true --type=boolean
```

Vérification (doit renvoyer une redirection `303` vers l'URL d'autorisation
Keycloak avec le bon `client_id` et `redirect_uri`) :

```bash
curl -I "https://nextcloud.sae.local/index.php/apps/user_oidc/login/1"
```

Test final : se connecter sur `https://nextcloud.sae.local` avec un compte
LDAP (`alice`/`ChangeMe123!`) via le bouton de connexion Keycloak.

## Phase 4 : Lstu

Installation officielle par clonage Git + Carton (gestionnaire de
dépendances Perl, hors Docker), suivant la [page d'installation du wiki
officiel](https://framagit.org/fiat-tux/hat-softwares/lstu/-/wikis/installation).
Base de données MariaDB (réutilisation de l'instance existante), authentification
déléguée à OAuth2 Proxy en Phase 5 (pas de `ldap`/`htpasswd` natifs de Lstu).

### 4.1 Dépendances système

Selon la [page "Dependencies"](https://framagit.org/fiat-tux/hat-softwares/lstu/-/wikis/Dependencies)
du wiki, plus les bibliothèques de développement MariaDB pour le backend MySQL :

```bash
sudo apt install -y build-essential libssl-dev zlib1g-dev libpng-dev carton \
  libmariadbd-dev libmariadb-dev-compat git
```

### 4.2 Base de données

```sql
CREATE DATABASE lstu CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'lstu'@'localhost' IDENTIFIED BY '<MOT_DE_PASSE_DB>';
GRANT ALL PRIVILEGES ON lstu.* TO 'lstu'@'localhost';
FLUSH PRIVILEGES;
```

### 4.3 Récupération des sources et installation des dépendances Perl

```bash
git clone https://framagit.org/fiat-tux/hat-softwares/lstu.git
cd lstu
```

Choix **MySQL** (compatible MariaDB) comme backend, sans LDAP ni htpasswd
natifs (délégués à OAuth2 Proxy), sans cache memcached (non nécessaire pour
ce TP) :

```bash
carton install --without=test --without=sqlite --without=postgresql \
  --without=ldap --without=htpasswd --without=cache
```

> Trois problèmes ont été rencontrés et résolus pendant l'installation :
>
> 1. **`Image::PNG::QRCode` ne compile pas sur Perl 5.40/GCC 14** (erreur
>    `incompatible pointer type` dans le code XS). Le `cpanfile.snapshot`
>    figé dans le dépôt officiel pointe vers la version 0.10 (2024), buguée
>    sur les toolchains récentes ; la version 0.11 (mai 2025) corrige le
>    problème. Correction avec la commande Carton documentée pour mettre à
>    jour un module précis sans tout re-résoudre :
>    ```bash
>    carton update Image::PNG::QRCode
>    ```
> 2. **`Net::Google::SafeBrowsing4` indisponible alors qu'il est marqué
>    "optionnel"** dans le `cpanfile` (groupe `feature 'safebrowsing'`).
>    En réalité, `lib/Lstu/Plugin/Helpers.pm` fait un `use
>    Net::Google::SafeBrowsing4;` à l'intérieur d'un bloc `if`, mais `use`
>    s'exécute toujours à la compilation en Perl, quel que soit le `if`.
>    Le module est donc chargé que l'option `safebrowsing_api_key` soit
>    configurée ou non. Il faut l'installer même sans vouloir utiliser la
>    fonctionnalité (`--without=safebrowsing` casse le démarrage) :
>    la commande `carton install` ci-dessus (sans ce `--without`) l'installe.
> 3. **`DBD::mysql` introuvable alors que `DBD::MariaDB` est installé.**
>    Le code de Lstu construit toujours une chaîne de connexion `mysql://`
>    (jamais `mariadb://`, même si `Mojo::mysql` sait gérer les deux), ce
>    qui force le chargement du pilote `DBD::mysql`. Installé via le paquet
>    Debian (il se lie très bien à `libmariadb`, aucune incompatibilité) :
>    ```bash
>    sudo apt install -y libdbd-mysql-perl
>    ```
>    Ce paquet s'installe dans l'arborescence Perl système ; l'environnement
>    isolé de Carton (`local/`) retombe dessus via `@INC`, sans conflit.

### 4.4 Déploiement

```bash
sudo mkdir -p /var/www/lstu
sudo rsync -a --exclude='.git' lstu/ /var/www/lstu/
sudo chown -R www-data:www-data /var/www/lstu
```

### 4.5 Configuration (`lstu.conf`)

```bash
cp lstu.conf.template lstu.conf
```

```perl
{
    hypnotoad => {
        listen => ['http://127.0.0.1:8081'],
        proxy  => 1,
    },

    contact       => 'admin@sae.local',

    secret        => ['<CHAINE_ALEATOIRE>'],

    hashed_adminpwd => '<SHA256_DU_MOT_DE_PASSE_ADMIN>',

    dbtype => 'mysql',

    mysqldb => {
        database => 'lstu',
        host     => 'localhost',
        user     => 'lstu',
        pwd      => '<MOT_DE_PASSE_DB>',
    },
};
```

Le hash du mot de passe admin (page `/stats`) se génère avec :

```bash
echo -n 'MotDePasseAdmin' | sha256sum
```

### 4.6 Service systemd

Fichier officiel `utilities/lstu.service`, seul le chemin de `carton` est
adapté (`/usr/bin/carton`, chemin du paquet Debian, au lieu de
`/usr/local/bin/carton` utilisé dans le template d'origine) :

```ini
[Unit]
Description=Shortened URLs service
Documentation=https://framagit.org/fiat-tux/hat-softwares/lstu
Requires=network.target
After=network.target mariadb.service

[Service]
Type=forking
User=www-data
RemainAfterExit=yes
WorkingDirectory=/var/www/lstu/
PIDFile=/var/www/lstu/script/hypnotoad.pid
ExecStart=/usr/bin/carton exec hypnotoad script/lstu
ExecStop=/usr/bin/carton exec hypnotoad -s script/lstu
ExecReload=/usr/bin/carton exec hypnotoad script/lstu

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now lstu.service
```

### 4.7 Vhost Nginx

Adapté du template officiel `utilities/lstu.nginx` (ajout du bloc TLS avec
le certificat wildcard, redirection HTTP→HTTPS) :

```nginx
upstream lstu {
    server 127.0.0.1:8081;
}

server {
    listen 80;
    listen [::]:80;
    server_name lstu.sae.local;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name lstu.sae.local;

    ssl_certificate     /etc/nginx/ssl/wildcard.sae.local.crt;
    ssl_certificate_key /etc/nginx/ssl/wildcard.sae.local.key;

    access_log  /var/log/nginx/lstu.access.log;
    error_log   /var/log/nginx/lstu.error.log;

    location / {
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Remote-Port $remote_port;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_pass http://lstu;
    }
}
```

> Le `proxy_redirect` du template officiel a été omis : comme `proxy => 1`
> est activé dans `lstu.conf`, Mojolicious détecte lui-même le reverse
> proxy via les en-têtes `X-Forwarded-*` et génère déjà des URLs absolues
> correctes, rendant cette réécriture inutile.

### 4.8 Cartographie des routes (utile pour la Phase 5)

| Route            | Rôle                                   | Doit être protégée ? |
|-------------------|-----------------------------------------|------------------------|
| `GET /`           | Formulaire de création de lien court    | Oui |
| `POST /a`         | Création effective du lien court (API/form) | Oui |
| `GET /:short`     | Redirection vers l'URL cible            | **Non** (public, imposé par le sujet) |

### 4.9 Vérification

```bash
curl -X POST https://lstu.sae.local/a -d 'lsturl=https://example.org' -d 'format=json'
# {"success":true,"short":"https://lstu.sae.local/XXXXXXXX","url":"https://example.org", ...}

curl -I https://lstu.sae.local/XXXXXXXX
# HTTP/2 301, location: https://example.org/
```

## Phase 5 : OAuth2 Proxy

Installation du binaire officiel (release GitHub, hors Docker), provider
`keycloak-oidc`, protection par Nginx `auth_request` des deux routes Lstu
identifiées en Phase 4 (`/` et `/a`), restriction au groupe LDAP
`lstu-admins` (créé en Phase 1).

### 5.1 Préparation côté Keycloak

Mapper d'audience (requis par la doc oauth2-proxy : le claim `aud` du
token doit contenir le `client_id`) :

```bash
/opt/keycloak/bin/kcadm.sh create clients/<ID_CLIENT_OAUTH2_PROXY>/protocol-mappers/models -r sae \
  -s name=audience-mapper \
  -s protocol=openid-connect \
  -s protocolMapper=oidc-audience-mapper \
  -s 'config."included.client.audience"=oauth2-proxy' \
  -s 'config."id.token.claim"=true' \
  -s 'config."access.token.claim"=true'
```

Client scope `groups` (optionnel, avec mapper d'appartenance aux
groupes) pour restreindre l'accès au groupe `lstu-admins` :

```bash
/opt/keycloak/bin/kcadm.sh create client-scopes -r sae \
  -s name=groups -s protocol=openid-connect \
  -s 'attributes."include.in.token.scope"=true' \
  -s 'attributes."display.on.consent.screen"=false' -i

/opt/keycloak/bin/kcadm.sh create client-scopes/<ID_SCOPE_GROUPS>/protocol-mappers/models -r sae \
  -s name=groups -s protocol=openid-connect \
  -s protocolMapper=oidc-group-membership-mapper \
  -s 'config."claim.name"=groups' \
  -s 'config."full.path"=true' \
  -s 'config."id.token.claim"=true' \
  -s 'config."access.token.claim"=true' \
  -s 'config."userinfo.token.claim"=true'

/opt/keycloak/bin/kcadm.sh update clients/<ID_CLIENT_OAUTH2_PROXY>/optional-client-scopes/<ID_SCOPE_GROUPS> -r sae
```

> Le scope `groups` est optionnel : il ne sera inclus dans le token que si
> le client le demande explicitement via le paramètre `scope`. D'où
> `scope = "openid email profile groups"` dans la configuration
> oauth2-proxy ci-dessous.

### 5.2 Installation du binaire

```bash
curl -sLO https://github.com/oauth2-proxy/oauth2-proxy/releases/download/v7.15.5/oauth2-proxy-v7.15.5.linux-amd64.tar.gz
curl -sLO https://github.com/oauth2-proxy/oauth2-proxy/releases/download/v7.15.5/oauth2-proxy-v7.15.5.linux-amd64.tar.gz-sha256sum.txt
sha256sum -c oauth2-proxy-v7.15.5.linux-amd64.tar.gz-sha256sum.txt

tar -xzf oauth2-proxy-v7.15.5.linux-amd64.tar.gz
sudo install -o root -g root -m 755 oauth2-proxy-v7.15.5.linux-amd64/oauth2-proxy /usr/local/bin/oauth2-proxy
```

### 5.3 Configuration (`/etc/oauth2-proxy/oauth2-proxy.cfg`)

```bash
sudo useradd -r -M -d /nonexistent -s /usr/sbin/nologin oauth2-proxy
sudo mkdir -p /etc/oauth2-proxy
```

Secret de cookie généré selon la méthode documentée (32 octets aléatoires,
encodage base64 url-safe) :

```bash
python3 -c 'import os,base64; print(base64.urlsafe_b64encode(os.urandom(32)).decode())'
```

```ini
provider = "keycloak-oidc"
client_id = "oauth2-proxy"
client_secret = "<SECRET_CLIENT_OAUTH2_PROXY>"
oidc_issuer_url = "https://keycloak.sae.local/realms/sae"
redirect_url = "https://lstu.sae.local/oauth2/callback"
code_challenge_method = "S256"

scope = "openid email profile groups"
oidc_groups_claim = "groups"
allowed_groups = [ "/lstu-admins" ]
email_domains = [ "*" ]

cookie_secret = "<SECRET_COOKIE_GENERE>"
cookie_domains = [ "lstu.sae.local" ]
cookie_secure = true

http_address = "127.0.0.1:4180"
reverse_proxy = true
trusted_proxy_ips = [ "127.0.0.1/32" ]

upstreams = [ "static://202" ]

session_store_type = "redis"
redis_connection_url = "redis://127.0.0.1:6379"
```

> Problème rencontré : `502 Bad Gateway` sur `/oauth2/callback`.
> Le token OIDC de Keycloak inclut le claim `groups` (nécessaire à la
> restriction par groupe ci-dessus), ce qui fait grossir le cookie de
> session au-delà de 4 Ko. oauth2-proxy le scinde alors en plusieurs
> `Set-Cookie`, et Nginx rejette la réponse
> (`upstream sent too big header while reading response header`).
>
> La documentation officielle recommande explicitement, pour ce cas précis,
> de stocker la session côté serveur avec **Redis** plutôt que de bricoler
> le découpage de cookies dans Nginx : seul un petit identifiant de session
> transite alors dans le cookie du navigateur.
>
> ```bash
> sudo apt install -y redis-server
> ```
>
> Installation Debian déjà restreinte à `127.0.0.1` par défaut (vérifié
> dans `/etc/redis/redis.conf`), cohérent avec le reste de l'infra
> (MariaDB, OpenLDAP) : aucun durcissement supplémentaire nécessaire.

> `trusted_proxy_ips` et non `trusted_ips` : ce sont deux options
> différentes. `trusted_ips` fait entièrement sauter l'authentification
> pour les IP listées (zéro contrôle), alors que `trusted_proxy_ips` indique
> seulement à oauth2-proxy quelles IP en amont sont autorisées à définir les
> en-têtes `X-Forwarded-*` quand `reverse_proxy = true`. C'est ce second
> comportement qu'il nous faut puisque Nginx tourne en local sur la même
> machine. Vérifié dans le code source officiel (`pkg/apis/options`).
>
> `upstreams = ["static://202"]` : on utilise exclusivement le mode
> `auth_request` de Nginx (voir plus bas), donc oauth2-proxy n'a pas besoin
> de relayer réellement les requêtes vers Lstu, juste de répondre "autorisé"
> ou "non autorisé" sur son endpoint `/oauth2/auth`.

### 5.4 Service systemd

Fichier officiel `contrib/oauth2-proxy.service.example` du dépôt, seul le
chemin du binaire est adapté (`/usr/local/bin/oauth2-proxy` au lieu de
`/usr/bin/oauth2-proxy`, correspondant à l'emplacement choisi en 5.2) :

```ini
[Unit]
Description=oauth2-proxy daemon service
After=network.target network-online.target nss-lookup.target basic.target keycloak.service nginx.service
Wants=network-online.target nss-lookup.target
StartLimitIntervalSec=30
StartLimitBurst=3

[Service]
User=oauth2-proxy
Group=oauth2-proxy
Restart=on-failure
RestartSec=30
WorkingDirectory=/etc/oauth2-proxy
ExecStart=/usr/local/bin/oauth2-proxy --config=/etc/oauth2-proxy/oauth2-proxy.cfg
ExecReload=/bin/kill -HUP $MAINPID
LimitNOFILE=65535
NoNewPrivileges=true
ProtectHome=true
ProtectSystem=full
ProtectHostname=true
ProtectControlGroups=true
ProtectKernelModules=true
ProtectKernelTunables=true
LockPersonality=true
RestrictRealtime=yes
RestrictNamespaces=yes
MemoryDenyWriteExecute=yes
PrivateDevices=yes
PrivateTmp=true
CapabilityBoundingSet=

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now oauth2-proxy.service
```

> Au démarrage de la VM, oauth2-proxy peut échouer une première fois
> (`502` lors de la découverte OIDC) si Keycloak n'a pas encore fini de
> démarrer : `After=keycloak.service` ordonne le démarrage après
> Keycloak, mais `systemd` considère ce dernier "démarré" dès que son
> processus est lancé, pas quand il répond réellement aux requêtes
> (environ 30s de plus). `Restart=on-failure` avec `RestartSec=30`
> suffit à rattraper ce décalage automatiquement au démarrage suivant.

### 5.5 Nginx : protection `auth_request`

Vhost Lstu complété (voir Phase 4 pour l'historique), à partir du motif
officiel documenté par oauth2-proxy
(`configuration/integrations/nginx`) : seules les routes identifiées en
4.8 comme permettant l'ajout d'une URL (`GET /` et `POST /a`) sont
protégées ; tout le reste, en particulier `GET /:short`, reste servi par
le `location /` générique, sans `auth_request`.

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name lstu.sae.local;

    ssl_certificate     /etc/nginx/ssl/wildcard.sae.local.crt;
    ssl_certificate_key /etc/nginx/ssl/wildcard.sae.local.key;

    access_log  /var/log/nginx/lstu.access.log;
    error_log   /var/log/nginx/lstu.error.log;

    location /oauth2/ {
        proxy_pass       http://127.0.0.1:4180;
        proxy_set_header Host                    $host;
        proxy_set_header X-Real-IP               $remote_addr;
        proxy_set_header X-Auth-Request-Redirect $request_uri;
    }

    location = /oauth2/auth {
        proxy_pass       http://127.0.0.1:4180;
        proxy_set_header Host             $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-Uri  $request_uri;
        proxy_set_header Content-Length   "";
        proxy_pass_request_body           off;
    }

    location @oauth2_signin {
        return 302 /oauth2/sign_in?rd=$scheme://$host$request_uri;
    }

    location = / {
        auth_request /oauth2/auth;
        error_page 401 = @oauth2_signin;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Remote-Port $remote_port;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_pass http://lstu;
    }

    location = /a {
        auth_request /oauth2/auth;
        error_page 401 = @oauth2_signin;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Remote-Port $remote_port;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_pass http://lstu;
    }

    location / {
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Remote-Port $remote_port;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_pass http://lstu;
    }
}
```

### 5.6 Vérification

```bash
# Page de création non authentifiée -> redirection vers Keycloak
curl -I https://lstu.sae.local/
# 302, Location: https://lstu.sae.local/oauth2/sign_in?rd=...

# Création non authentifiée -> bloquée
curl -I -X POST https://lstu.sae.local/a -d 'lsturl=https://example.com'
# 302 (redirection vers la connexion, pas de création)

# Lien court existant -> toujours public
curl -I https://lstu.sae.local/<code_court_existant>
# 301 vers l'URL cible, aucune authentification demandée
```

Test navigateur de bout en bout : `bob` (membre de `lstu-admins`) peut se
connecter et raccourcir une URL ; `alice` (hors groupe) se voit refuser
l'accès (403) après authentification réussie auprès de Keycloak : la
restriction par groupe fonctionne donc à un niveau différent (et
complémentaire) de l'authentification elle-même.

> Dépannage : en cas de comportement anormal après un changement de
> configuration (login qui boucle, `502`, session qui semble "coincée"),
> commencer par vider les cookies du navigateur pour `lstu.sae.local`
> avant de chercher plus loin. Le cookie `_oauth2_proxy` peut rester
> volumineux ou invalide suite à une configuration précédente (ex: avant la
> mise en place de Redis ci-dessus), et un cookie périmé peut masquer ou
> imiter d'autres problèmes.

## Phase 6 : Matomo (bonus)

Installation officielle (archive zip signée GPG, hors Docker), PHP-FPM +
MariaDB (réutilisation de l'infra existante), suivi des visites du site
Lstu via l'intégration native de Lstu (`Mojolicious::Plugin::Piwik`, déjà
présente dans ses dépendances Carton).

### 6.1 Prérequis

Toutes les extensions PHP requises par Matomo (`pdo_mysql`, `curl`, `gd`,
`mbstring`, `xml`, `cli`) sont déjà installées depuis la Phase 3
(Nextcloud). Aucun paquet supplémentaire nécessaire.

### 6.2 Base de données

```sql
CREATE DATABASE matomo CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'matomo'@'localhost' IDENTIFIED BY '<MOT_DE_PASSE_DB>';
GRANT ALL PRIVILEGES ON matomo.* TO 'matomo'@'localhost';
FLUSH PRIVILEGES;
```

### 6.3 Téléchargement, vérification et déploiement

```bash
curl -sLO https://builds.matomo.org/matomo.zip
curl -sLO https://builds.matomo.org/matomo.zip.asc

gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys F529A27008477483777FC23D63BB30D0E5D2C749
gpg --verify matomo.zip.asc matomo.zip
# doit afficher "Good signature from Matomo <hello@matomo.org>"

unzip -q matomo.zip
sudo mkdir -p /var/www/matomo
sudo rsync -a matomo/ /var/www/matomo/
sudo chown -R www-data:www-data /var/www/matomo
sudo chmod -R u+rwX,go+rX,go-w /var/www/matomo
```

### 6.4 Vhost Nginx

Repris quasi à l'identique du dépôt officiel
[`matomo-org/matomo-nginx`](https://github.com/matomo-org/matomo-nginx)
(seuls le domaine, le certificat et le socket PHP-FPM sont adaptés) :

```nginx
server {
    listen [::]:80;
    listen 80;
    server_name matomo.sae.local;
    location / {
        return 301 https://$host$request_uri;
    }
}
server {
    listen [::]:443 ssl;
    listen 443 ssl;
    http2 on;
    server_name matomo.sae.local;
    access_log /var/log/nginx/matomo.access.log;
    error_log /var/log/nginx/matomo.error.log;

    ssl_certificate     /etc/nginx/ssl/wildcard.sae.local.crt;
    ssl_certificate_key /etc/nginx/ssl/wildcard.sae.local.key;

    add_header Referrer-Policy origin always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    root /var/www/matomo/;
    index index.php;

    location ~ ^/(index|matomo|piwik|js/index|plugins/HeatmapSessionRecording/configs)\.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_param HTTP_PROXY "";
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }

    location ~* ^.+\.php$ {
        deny all;
        return 403;
    }

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ ^/(config|tmp|core|lang) {
        deny all;
        return 403;
    }

    location ~ /\.ht {
        deny  all;
        return 403;
    }

    location ~ js/container_.*_preview\.js$ {
        expires off;
        add_header Cache-Control 'private, no-cache, no-store';
    }

    location ~ \.(gif|ico|jpg|png|svg|js|css|htm|html|mp3|mp4|wav|ogg|avi|ttf|eot|woff|woff2)$ {
        allow all;
        expires 1h;
        add_header Pragma public;
        add_header Cache-Control "public";
    }

    location ~ ^/(libs|vendor|plugins|misc|node_modules) {
        deny all;
        return 403;
    }

    location ~/(.*\.md|LEGALNOTICE|LICENSE) {
        default_type text/plain;
    }
}
```

> La ligne `try_files $fastcgi_script_name =404;` du bloc officiel a été
> omise dans le premier `location` : le snippet Debian
> `/etc/nginx/snippets/fastcgi-php.conf` (inclus via `include
> snippets/fastcgi-php.conf;`) la définit déjà, et Nginx refuse une
> directive dupliquée (`nginx: [emerg] "try_files" directive is duplicate`).

### 6.5 Installation (assistant web uniquement)

Contrairement à Nextcloud, Matomo ne propose aucun installeur en ligne de
commande officiellement documenté (vérifié : `console list` ne liste
aucune commande `core:install` ou équivalent). La seule méthode officielle
est l'assistant web, à dérouler sur `https://matomo.sae.local/` :

1. Welcome → Next
2. System Check → Next
3. Base MySQL : serveur `localhost`, login `matomo`, mot de passe, base
   `matomo`, préfixe `matomo_`
4. Création des tables → Next
5. Compte Superuser (login + mot de passe à conserver)
6. Premier site suivi : nom `Lstu SAE`, URL `https://lstu.sae.local/`
7. Code de tracking fourni par l'assistant (voir 6.6)

### 6.6 Intégration avec Lstu (démonstration d'usage)

Lstu embarque déjà `Mojolicious::Plugin::Piwik` (chargé sans condition
dans `lib/Lstu.pm`, `$self->plugin('Piwik')`) et expose l'assistant
`piwik_tag` dans tous ses templates. La variante "suivi serveur à chaque
redirection de lien court" nécessiterait d'activer Minion (volontairement
écarté en Phase 4) ; on utilise donc le suivi JavaScript classique,
exactement comme préconisé par l'assistant d'installation de Matomo
("coller avant `</head>`"), en l'insérant une fois dans le layout du thème
plutôt que sur chaque page manuellement :

```bash
sudo sed -i "s|    </head>|        %= piwik_tag 1, 'matomo.sae.local'\n    </head>|" \
  /var/www/lstu/themes/default/templates/layouts/default.html.ep
sudo systemctl reload-or-restart lstu
```

(`1` = ID du site créé à l'étape 6.5 ; `matomo.sae.local` = domaine de
l'instance Matomo, avec les mêmes paramètres que le code JS généré par
l'assistant.)

> Problème rencontré : aucune visite enregistrée dans Matomo, et aucune
> requête `matomo.php`/`matomo.js` dans les logs Nginx, même sans
> bloqueur de publicité actif. Cause : Lstu envoie par défaut un en-tête
> `Content-Security-Policy` strict (`default-src 'none'; script-src
> 'self'; img-src 'self' data:; ...`, voir `lstu.conf.template`), qui
> interdit au navigateur de charger `matomo.js` depuis un autre domaine ou
> d'envoyer la requête de tracking. Le blocage est silencieux côté
> navigateur, donc rien n'atteint le serveur (ni succès, ni erreur
> visible). La documentation du plugin `Mojolicious::Plugin::Piwik`
> anticipe justement ce cas ("to embed the tracking code in pages with
> strict Content Security Policies... add the domain of your Matomo
> instance to your CSP as a valid script-src and potentially img-src").
> Première correction via l'option `csp` de `lstu.conf` (déjà documentée
> dans le template, seulement commentée par défaut), en ajoutant le
> domaine Matomo à `script-src`/`img-src`/`connect-src` :
>
> ```perl
> csp => "default-src 'none'; script-src 'self' https://matomo.sae.local; style-src 'self'; img-src 'self' data: https://matomo.sae.local; font-src 'self'; form-action 'self'; base-uri 'self'; connect-src 'self' https://matomo.sae.local",
> ```
>
> Insuffisant : la console navigateur montrait encore un blocage CSP,
> cette fois `"Executing inline script violates ... 'script-src'"`. Cause
> réelle : le tag généré par `piwik_tag` est un `<script>` **inline** (pas
> un fichier externe), et lister un domaine dans `script-src` n'autorise
> que le chargement de scripts externes depuis ce domaine, jamais
> l'exécution de code inline. Il faut soit `'unsafe-inline'` (trop
> permissif : autorise n'importe quel script inline sur toute la page),
> soit le hash exact du script, solution plus propre retenue ici, le
> navigateur fournissant lui-même le hash attendu dans son message
> d'erreur :
>
> ```perl
> csp => "default-src 'none'; script-src 'self' https://matomo.sae.local 'sha256-LG3oMj0OdCdDnjVCl5Rmh+J9AxNtuf4qmuwLXvjsJ14='; style-src 'self'; img-src 'self' data: https://matomo.sae.local; font-src 'self'; form-action 'self'; base-uri 'self'; connect-src 'self' https://matomo.sae.local",
> ```
>
> ```bash
> sudo systemctl reload-or-restart lstu
> ```
>
> Pour fiabiliser la valeur du hash (des caractères comme `I`/`1` ou
> `O`/`0` sont faciles à confondre lors d'une recopie manuelle), le plus
> sûr est de le recalculer directement à partir du contenu réellement
> servi plutôt que de le recopier à la main :
>
> ```bash
> curl -s https://lstu.sae.local/stats \
>   | grep -oPz '(?<=<script type="text/javascript">)[\s\S]*?(?=</script>)' \
>   | head -c -1 | sha256sum -b | cut -d' ' -f1 | xxd -r -p | base64
> ```
>
> Ce hash change si le contenu du tag (`site_id`, domaine Matomo, etc.)
> change : à régénérer après toute modification de la ligne `piwik_tag`
> dans le thème.

Vérifié de bout en bout via les logs Nginx : chargement de `matomo.js`
(200) suivi d'un `POST /matomo.php` (204) contenant
`url=https://lstu.sae.local/stats`. La visite remonte bien dans Matomo,
sous Visitors → Visits Log.

### 6.7 Vérification

```bash
curl -s http://127.0.0.1:8081/stats | grep -A5 piwik_tag
```

Doit afficher un bloc `<script>` avec `setTrackerUrl` pointant vers
`https://matomo.sae.local/matomo.php` et `setSiteId` à `1`.

Test final : visiter une page Lstu (ex: `https://lstu.sae.local/stats`)
dans un navigateur, puis consulter **Matomo → Visitors → Visits Log**
pour confirmer qu'une nouvelle visite est enregistrée.
