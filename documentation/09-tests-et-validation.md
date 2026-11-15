# 10 — Tests et validation

## 1. Objectif

Cette section rassemble les tests fonctionnels utilisés pour vérifier les principaux services du laboratoire.

## 2. Connectivité locale et inter-sites

Depuis le client Paris :
```cmd
ipconfig /all
ping 192.168.20.1
ping 192.168.10.20
```

Résultat attendu :
- passerelle locale joignable ;
- serveur du site Amiens joignable si ICMP est autorisé ;
- adresse, masque, passerelle et DNS cohérents avec le réseau Paris.

Côté pfSense, `Status > IPsec` montre le tunnel **Established** et la phase 2 **Installed** (voir [03](03-vpn-ipsec.md)).

## 3. DNS et domaine

```cmd
nslookup srv-ad.lab.local 192.168.10.20
whoami
echo %USERDOMAIN%
gpresult /r
```

Résultat observé : le client Paris a pu résoudre le serveur AD/DNS et rejoindre le domaine `lab.local`.

## 4. Partages réseau

Depuis l'explorateur Windows ou la boîte Exécuter :
```text
\\srv-ad.lab.local\Partages
```

Résultat observé :
- accès au partage parent depuis Paris ;
- visibilité des dossiers de service ;
- ouverture autorisée uniquement pour le dossier correspondant aux droits de l'utilisateur.

Les droits de partage et NTFS ont été relevés avec `Get-SmbShareAccess` et `icacls` (voir [08](08-partages-et-permissions.md)).

## 5. Mappage des lecteurs

Actualiser les stratégies :
```cmd
gpupdate /force
```

Vérifier dans l'explorateur les lecteurs correspondant au groupe de l'utilisateur :
- `I:` pour `GG_IT` ;
- `R:` pour `GG_RH` ;
- `T:` pour `GG_Direction`.

Les tests réalisés avec les comptes des différents services ont confirmé le fonctionnement attendu.

## 6. Politique de mots de passe et de verrouillage

```cmd
gpupdate /force
net accounts /domain
```

Résultat observé après correction de l'ordre des liens de GPO : les valeurs configurées (longueur minimale 8, âge maximal 90 jours, seuil de verrouillage 3, durée de verrouillage 30 minutes) sont appliquées. Avant la correction, `net accounts /domain` affichait les valeurs par défaut de Windows (7 caractères, 42 jours, pas de verrouillage), parce que la `Default Domain Policy` était prioritaire (voir [06](06-gpo-et-administration.md) et [11](11-evolutions-et-lecons.md)).

## 7. Tableau de synthèse

| Fonctionnalité | État |
|---|---|
| Réseaux locaux Amiens et Paris | Configurés |
| VPN IPsec site-à-site | Configuré, tunnel établi, trafic observé dans les deux sens |
| Communication Paris vers serveur Amiens | Testée |
| Active Directory / DNS | Opérationnels dans le périmètre testé |
| Client Paris joint au domaine | Validé |
| DHCP Amiens (`isc-dhcp-server` sur `srv-dhcp`) | Opérationnel : clients Windows et Linux servis |
| DHCP Paris par pfSense | Configuré, client servi (`192.168.20.105`) |
| GPO (10 personnalisées) | Appliquées sur les machines et comptes testés (`gpresult /r`) |
| Politique de mots de passe et de verrouillage | Appliquée après correction de la priorité des GPO |
| Profils itinérants | Configurés et testés dans le laboratoire |
| Partages SMB et permissions NTFS | Testés ; droits relevés avec `Get-SmbShareAccess` et `icacls` |
| Mappage des lecteurs par groupes | Testé avec les comptes des services |

## 8. Limites identifiées

- Un seul contrôleur de domaine est présent dans le périmètre documenté.
- La haute disponibilité AD/DNS n'est pas mise en œuvre.
- Les règles de pare-feu entre les deux LAN autorisent tous les protocoles et tous les ports (laboratoire).
- Le partage `Profils$` est en contrôle total pour Tout le monde ; la protection repose sur les droits NTFS.
- Les paramètres détaillés de la GPO `GPO-ProfilsItinerants` ne sont pas documentés.
- Les sauvegardes et procédures de restauration ne sont pas décrites comme validées.

## 9. Tests spécifiques DHCP et GPO

Les procédures de test détaillées sont dans [05 — DHCP](05-dhcp.md) (section 5) et [06 — GPO](06-gpo-et-administration.md) (section 8), en particulier `ipconfig /all`, `gpresult /r` et `net accounts /domain`.

## 10. Captures

Les captures du laboratoire sont intégrées dans les documents correspondants :
- réseau et virtualisation : [01](01-architecture-reseau.md) ;
- pfSense et pare-feu : [02](02-pfsense-et-routage.md) ;
- tunnel IPsec : [03](03-vpn-ipsec.md) ;
- Active Directory, groupes et DNS : [04](04-active-directory-et-dns.md) ;
- DHCP : [05](05-dhcp.md) ;
- GPO, `gpresult` et lecteurs mappés : [06](06-gpo-et-administration.md) ;
- permissions de partage et NTFS : [08](08-partages-et-permissions.md).
