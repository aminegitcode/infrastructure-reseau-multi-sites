# 11 — Évolutions et leçons

Ce document retrace les décisions prises pendant la construction du laboratoire et les problèmes rencontrés, avec leur résolution.

## 1. Évolution majeure : du DNS BIND9 au DNS Active Directory

| Étape | Situation |
|---|---|
| Départ | Un serveur Ubuntu cumulait DHCP et DNS (BIND9) et distribuait sa propre adresse comme DNS |
| Problème | La jonction d'un poste Windows au domaine échouait : « impossible de contacter un contrôleur de domaine Active Directory » |
| Cause | BIND9 ne contenait pas les enregistrements SRV d'Active Directory, indispensables pour localiser un contrôleur de domaine |
| Décision | Confier le DNS de `lab.local` à `srv-ad` (le DNS intégré à Active Directory) |
| Résultat | Le DHCP distribue `192.168.10.20`, la VM Linux est réinstallée sous le nom `srv-dhcp` et ne fait plus que du DHCP ; la jonction au domaine fonctionne |

**À retenir :** dans un domaine Active Directory, le DNS n'est pas un service annexe, c'est la base de la découverte des services. Séparer les rôles (DHCP d'un côté, AD et DNS de l'autre) est aussi une organisation courante en entreprise.

## 2. Passage d'un site unique à une architecture multi-sites

Le laboratoire a d'abord été conçu sur un seul réseau (`192.168.10.0/24`). Il a ensuite été étendu avec :
- un second site (Paris, `192.168.20.0/24`) ;
- deux pare-feu pfSense, un par site ;
- un VPN IPsec site-à-site ;
- un DHCP local à Paris, assuré par pfSense, qui distribue le DNS d'Active Directory d'Amiens.

## 3. Problèmes rencontrés et solutions

| Domaine | Problème | Cause probable / diagnostic | Solution |
|---|---|---|---|
| Virtualisation | Une VM tente un démarrage réseau (PXE) ou signale qu'aucun système n'est trouvé | Support d'installation ou disque non pris en compte au démarrage | Vérifier l'ISO monté et l'ordre de démarrage dans VMware ; réinstaller si besoin |
| Réseau Linux | Interface `ens33` à l'état `DOWN` | Interface non configurée ou non activée | Configuration Netplan (adresse statique), application, puis vérification avec `ip a` |
| Réseau Linux | Pas d'accès à Internet pour installer un paquet | Route par défaut incorrecte ; le LAN Segment est isolé | Carte NAT temporaire, route par défaut corrigée, puis retour sur le LAN Segment |
| Paquets | Messages d'avertissement sur des sources `apt` en double | Entrée déclarée deux fois | Suppression du doublon |
| DHCP | Service `isc-dhcp-server` en échec | Faute de frappe : `max-lease-tim` au lieu de `max-lease-time` | Correction, puis validation avec `dhcpd -t -cf` avant redémarrage |
| DNS (BIND9) | Service non valide | `;` manquant dans `named.conf.local` | Correction de la syntaxe (service ensuite abandonné au profit du DNS d'AD) |
| Active Directory | Promotion en contrôleur de domaine bloquée | Mot de passe de l'administrateur local vide | Définition d'un mot de passe pour le compte administrateur local |
| Domaine | Jonction impossible | DNS distribué sans enregistrements SRV | DNS d'Active Directory distribué par DHCP (section 1) |
| DHCP | Un client Windows affichait `192.168.10.20` (l'adresse de `srv-ad`) | Carte réseau restée en adresse statique | « Obtenir une adresse IP automatiquement », puis `ipconfig /release` et `/renew` |
| GPO | `net accounts /domain` affichait 7 caractères, 42 jours et aucun verrouillage malgré `GPO-PasswordPolicy` et `GPO-AccountLockout` | La `Default Domain Policy` était prioritaire dans l'ordre des liens du domaine | Montée des deux GPO au-dessus de la `Default Domain Policy`, puis `gpupdate /force` : 8 caractères, 90 jours, 3 tentatives, 30 minutes |
| Sécurité des partages | `C:\Wallpapers` en contrôle total NTFS pour Tout le monde | Droits trop larges par rapport au besoin (lecture) | Retrait de « Tout le monde » et lecture et exécution pour les utilisateurs du domaine |
| GPO | Fond d'écran non appliqué | Chemin local au serveur (`C:\Wallpapers`) introuvable côté client | Dossier partagé `\\srv-ad\Wallpapers` en chemin UNC, permissions de partage et NTFS en lecture |


## 4. Leçons

- **Valider avant de redémarrer** : `dhcpd -t` détecte les erreurs de syntaxe avant qu'elles ne cassent le service.
- **Lire les journaux** : `systemctl status` et `journalctl -u <service>` indiquent presque toujours la ligne en cause.
- **Un chemin pour un client est un chemin vu depuis le client** : tout ce qui est distribué par GPO doit être joignable en UNC par les postes.
- **Le DNS conditionne Active Directory** : vérifier d'abord quel serveur DNS un poste utilise avec `ipconfig /all`.
- **Partage et NTFS se cumulent** : l'accès effectif est le plus restrictif des deux.
- **Moindre privilège** : lecture seule sur les partages de ressources statiques, droits donnés à des groupes.
- **Documenter pendant la construction** : les captures et les commandes sont plus faciles à relever sur le moment qu'après coup.

## 5. Pistes d'amélioration

- Ajouter un second contrôleur de domaine (redondance AD/DNS).
- Mettre en place des sauvegardes et tester la restauration.
- Superviser les services (journaux, état du tunnel IPsec).
- Restreindre les règles de pare-feu entre les deux LAN aux flux nécessaires (DNS, Kerberos, LDAP, SMB, ICMP de diagnostic) au lieu de tout autoriser.
- Utiliser une clé prépartagée longue et aléatoire pour IPsec, la renouveler, ou passer à des certificats.
- Limiter le partage `Profils$` aux utilisateurs du domaine et durcir les droits hérités de `C:\Partages`.
- Migrer le DHCP de pfSense hors d'ISC DHCP, signalé en fin de vie par pfSense.
- Ajouter un DHCP de secours ou un mécanisme de redondance sur le site d'Amiens.
