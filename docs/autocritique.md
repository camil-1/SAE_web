# Auto-critique

Ce document revient sur les difficultés rencontrées pendant le montage de
l'infrastructure, et sur ce que nous ferions différemment si c'était à
refaire. Il se concentre sur la partie du travail suivie en détail sur
une des VM du groupe ; des difficultés propres aux autres VM ont pu ne
pas être consignées ici.

## Difficultés techniques

### Dépendances Perl de Lstu

L'installation de Lstu a été la phase la plus longue à déboguer, pour des
raisons qui n'avaient rien à voir avec notre configuration :

- Le `cpanfile.snapshot` figé dans le dépôt officiel pointait vers une
  version d'`Image::PNG::QRCode` (0.10) qui ne compile plus sur un Perl
  et un GCC récents (incompatibilité de type dans le code XS). Il a fallu
  descendre jusqu'aux logs de compilation `cpanm` pour comprendre que le
  problème ne venait pas de notre système mais d'une dépendance figée
  obsolète, puis trouver la bonne commande Carton pour ne mettre à jour
  que ce module précis.
- `Net::Google::SafeBrowsing4` est présenté comme une fonctionnalité
  optionnelle dans le `cpanfile`, mais le code source de Lstu
  (`Lstu/Plugin/Helpers.pm`) le charge avec un `use` placé à l'intérieur
  d'un `if`, ce qui ne le rend pas conditionnel en Perl : `use` s'exécute
  toujours à la compilation. La seule documentation ne permettait pas de
  comprendre ce point, il a fallu lire le code source.
- Le choix MySQL/MariaDB a buté sur une confusion entre les pilotes
  `DBD::mysql` et `DBD::MariaDB`, le code de Lstu imposant implicitement
  le premier.

Si c'était à refaire, nous irions lire le code source de Lstu (pas
seulement sa documentation ou son wiki) dès qu'une erreur de dépendance
semble étrange, plutôt que de supposer une erreur de notre côté. Le wiki
officiel ne couvre pas ces cas ; un coup d'œil au dépôt aurait permis
d'anticiper une partie de ces problèmes avant même de lancer
l'installation.

### Fédération LDAP → Keycloak incomplète (groupes manquants)

La fédération LDAP configurée dans Keycloak importait correctement les
utilisateurs, mais pas leurs groupes : un composant séparé
(`group-ldap-mapper`) est nécessaire en plus du composant `ldap`
lui-même. Cet oubli n'a été détecté qu'au moment de tester
concrètement la restriction d'accès par groupe sur OAuth2 Proxy (un
utilisateur pourtant membre du bon groupe LDAP se faisait refuser l'accès
partout), pas en configurant la fédération elle-même, qui ne signalait
aucune erreur.

Si c'était à refaire, nous testerions le chemin complet (LDAP → Keycloak
→ groupes dans le token) dès la mise en place de la fédération, avec un
utilisateur de test et une vérification explicite de ses groupes
(`kcadm.sh get users/.../groups`), plutôt que d'attendre la phase OAuth2
Proxy pour découvrir le trou. Une configuration qui « a l'air de marcher »
(utilisateurs importés, authentification réussie) peut cacher une
fonctionnalité en échec total et silencieux.

### Cookie de session trop volumineux (OAuth2 Proxy + Nginx)

L'ajout du claim `groups` au token OIDC (nécessaire à la restriction par
groupe) a fait dépasser au cookie de session la limite de 4 Ko, ce que
Nginx ne gère pas bien par défaut avec `auth_request` (`502 Bad Gateway`
sans message clair côté client). La solution, un stockage de session
côté serveur avec Redis plutôt qu'un découpage de cookies côté Nginx, est
bien documentée côté OAuth2 Proxy, mais seulement dans une partie assez
annexe de sa documentation (la page d'intégration Nginx) : facile à
manquer tant qu'on n'a pas le nez dessus.

Si c'était à refaire, nous anticiperions ce point dès qu'un realm
Keycloak expose un claim `groups` (ou tout autre claim volumineux) à un
client OAuth2 Proxy placé derrière un `auth_request` Nginx, plutôt que de
le découvrir via un `502` en bout de chaîne.

### Content-Security-Policy de Lstu bloquant Matomo

Lstu envoie par défaut un en-tête CSP strict qui bloque silencieusement
(côté navigateur, sans aucune trace côté serveur) le chargement du script
de tracking Matomo. Le diagnostic a demandé de passer des logs serveur
(qui ne montraient rien d'anormal) à la console du navigateur, qui seule
révélait le blocage.

Si c'était à refaire, nous prendrions ce réflexe plus tôt : quand une
requête réseau attendue n'apparaît dans aucun log serveur, vérifier la
console du navigateur (CSP, mixed content, erreurs JS) avant de chercher
du côté serveur, en particulier pour toute intégration client (scripts
tiers, tracking, SSO) ajoutée sur une application qui a sa propre
politique de sécurité.

## Difficultés d'organisation

Le choix de travailler chacun sur sa propre copie de VM (détaillé dans
`methodologie.md`) a évité les conflits de configuration, mais au prix
d'une duplication de travail et de l'impossibilité de comparer facilement
l'état exact des trois infrastructures à un instant donné. Aucune des
solutions envisageables dans le temps imparti (VM partagée avec créneaux,
outils de gestion de configuration type Ansible) n'a été mise en place.

Si c'était à refaire, décrire l'infrastructure sous forme de scripts ou
de playbooks (même basiques) dès le début, plutôt qu'en configuration
manuelle, aurait permis de partager et de comparer le travail entre
membres du groupe sans dépendre d'une VM unique partagée. Le tutoriel
final s'en serait trouvé plus proche d'une exécution automatisée
rejouable, plutôt que d'une suite de commandes reconstituée après coup.

## Ce que nous referions à l'identique

- Suivre les sources officielles (dépôts Git, wiki, documentation) à la
  lettre plutôt que des tutoriels tiers, y compris quand cela demande de
  lire du code source : c'est la méthode qui a permis de comprendre
  plusieurs bugs amont plutôt que de les contourner à l'aveugle.
- Tester chaque brique jusqu'au bout (authentification réelle, pas
  seulement "le service démarre") avant de passer à la suivante : c'est
  ce qui a permis de détecter le problème de groupes LDAP et le blocage
  CSP, qui ne se seraient jamais révélés à la simple lecture des fichiers
  de configuration.
