# Méthodes de travail

## Composition du groupe

Le groupe Noa, Camil et Alix

## Répartition des tâches

Le travail a été réparti par phases et par logiciels entre les 3 membres
du groupe, chacun sur sa propre copie de la VM Debian 13 fournie.

## Organisation

Chaque membre a travaillé sur sa propre copie de la VM, montée
indépendamment via virt-manager, plutôt que sur une VM unique partagée.
Nous n'avons pas trouvé de meilleure solution dans le temps imparti : les
briques de l'infrastructure (OpenLDAP, Keycloak, Nextcloud, Lstu, OAuth2
Proxy) sont fortement interdépendantes, et leur configuration change en
continu pendant le montage (fichiers de service, bases de données,
certificats, realms Keycloak...). Faire avancer les 3 membres en
parallèle sur une VM unique aurait demandé de fusionner ces états à
chaque étape : plus compliqué, et plus risqué, que de dupliquer la VM
(conflits de configuration, services cassés par une modification
concurrente).

Le planning a suivi l'ordre imposé par le sujet : OpenLDAP, Keycloak et
Nextcloud d'abord, puis un Lstu fonctionnel, puis OAuth2 Proxy, et enfin
Matomo en bonus, le temps le permettant.

## Outils utilisés

- Virtualisation : virt-manager (QEMU/KVM), une copie de VM par membre du
  groupe.
- Communication : SMS, Signal et Discord.
- Partage des fichiers de configuration et de la documentation : dépôt
  Git du projet.
