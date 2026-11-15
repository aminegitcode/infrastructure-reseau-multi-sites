# 07 — Profils itinérants

## 1. Objectif

Un profil itinérant permet à un utilisateur de retrouver son environnement (bureau, fond d'écran, documents, paramètres) lorsqu'il ouvre une session sur n'importe quel poste du domaine, sous réserve de la configuration du profil et de la compatibilité des postes.

Dans ce laboratoire, les profils sont hébergés sur le serveur `srv-ad`.

## 2. Partage et chemin de profil

| Élément | Valeur |
|---|---|
| Dossier sur le serveur | `C:\Profils` |
| Partage | `Profils$` (partage masqué : le suffixe `$` le rend invisible lors d'une navigation classique) |
| Permissions de partage | Tout le monde : Contrôle total |
| Permissions NTFS | Héritées de `C:\` : Système et Administrateurs en contrôle total, Utilisateurs en lecture et création, Créateur propriétaire en contrôle total |
| Exemple de chemin de profil observé | `\\srv-ad\Profils$\administrateur.V6` |

Le chemin observé provient de `gpresult /r` : `Profil itinérant : \\srv-ad\Profils$\administrateur.V6`. Sur un poste client, le même relevé donne `\\srv-ad\Profils$\<compte>.V6` pour le compte connecté. Le suffixe `.V6` est une particularité de la version du profil Windows ; il ne faut pas le généraliser à tous les profils.

## 3. GPO associée

Une GPO nommée `GPO-ProfilsItinerants` est liée au domaine et apparaît dans les paramètres ordinateur de `srv-ad` d'après `gpresult /r` (voir [06 — GPO](06-gpo-et-administration.md), section 4.7).

## 4. Principe

1. L'utilisateur ouvre une session sur un poste membre du domaine.
2. Windows récupère le profil depuis le chemin réseau configuré.
3. Les paramètres pris en charge sont chargés dans la session.
4. À la fermeture de session, les changements du profil sont synchronisés vers le partage, selon les règles Windows et les éventuels conflits.

## 5. Vérifications

- Vérifier le chemin de profil dans les propriétés du compte utilisateur AD, ou avec `gpresult /r`.
- Vérifier que le partage est accessible.
- Tester l'ouverture de session sur un autre poste du domaine.
- Créer un changement de test non critique et vérifier sa persistance après reconnexion.
- Consulter les journaux Windows en cas d'erreur de chargement ou de synchronisation.


