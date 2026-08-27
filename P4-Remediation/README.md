# P4 — Remédiation et Hardening

> *Trouver une faille sans la corriger, c'est faire la moitié du travail. La moins utile.*

---

## Objectif de cette phase

Un audit qui s'arrête à la liste des vulnérabilités est incomplet. Le vrai travail, c'est ce qui vient après : **corriger, durcir, et documenter** pour que le réseau soit dans un état meilleur à la fin qu'au début.

Cette phase prend chaque vulnérabilité identifiée en P3 et lui applique une réponse concrète. On ne va pas juste "désactiver UPnP" — on explique pourquoi, on montre comment, et on vérifie que ça a fonctionné.

---

## Principe directeur : le moindre privilège

La règle de base du hardening est simple : **ce qui n'est pas nécessaire est supprimé**. Pas mis en veille, pas configuré pour "ne pas déranger" — supprimé ou désactivé. Chaque service actif, chaque port ouvert, chaque compte utilisateur est une surface d'attaque potentielle. On réduit la surface au strict minimum fonctionnel.

> 💡 **Hardening** (durcissement) : processus de sécurisation d'un système par la réduction de sa surface d'exposition. Désactiver les services inutiles, fermer les ports non utilisés, appliquer les mises à jour, renforcer les configurations par défaut.

---

## Remédiation 1 — Caméra IP : cas critique

### Problème
Identifiants par défaut, firmware non patché (CVE-2017-8225), port RTSP exposé sur Internet via UPnP.

### Actions

**1. Changer les identifiants immédiatement**

```bash
# Accès à l'interface web de la caméra
curl -X POST http://192.168.1.34/set_users.cgi \
  --data "user=admin&oldpasswd=admin&passwd=Str0ngP@ss2024&group=admin"
```

Ou via l'interface web : `http://192.168.1.34` → Paramètres → Utilisateurs → Modifier admin.

Le mot de passe doit respecter les critères minimaux : 12 caractères, majuscules, minuscules, chiffres, caractères spéciaux.

**2. Vérifier et appliquer les mises à jour firmware**

```bash
# Identifier la version firmware actuelle
curl -s http://192.168.1.34/get_status.cgi | grep firmware
```

Résultat : `firmware_version=2.4.1.2_180321` — version de mars 2018. Consulter le site du fabricant pour la dernière version disponible et appliquer la mise à jour via l'interface d'administration.

**3. Désactiver l'accès depuis Internet**

La règle UPnP qui expose le port 554 doit être supprimée. Deux façons :

```bash
# Supprimer la règle UPnP depuis la machine Linux
upnpc -d 554 TCP
```

Puis désactiver UPnP globalement sur la box (voir section Gateway ci-dessous).

**4. Isoler la caméra dans un VLAN dédié IoT**

Idéalement, une caméra IP ne devrait pas être sur le même réseau que les PC. Un VLAN (Virtual LAN — réseau local virtuel) permet de créer une isolation logique même sur un réseau physique unique.

```
VLAN 1 (LAN principal) : PC, NAS, Raspberry Pi  → 192.168.1.0/24
VLAN 2 (IoT isolé)     : Caméra, Echo, Chromecast → 192.168.2.0/24
```

La règle : les appareils du VLAN IoT peuvent sortir vers Internet mais **ne peuvent pas initier de connexions vers le VLAN principal**. Si une caméra est compromise, l'attaquant reste bloqué dans le VLAN IoT — il ne peut pas se déplacer vers les machines sensibles.

> 💡 **Mouvement latéral** (lateral movement) : technique par laquelle un attaquant, après avoir compromis un premier équipement, cherche à se propager vers d'autres machines du même réseau. L'isolation par VLAN limite cette propagation.

---

## Remédiation 2 — Gateway : désactivation d'UPnP

### Problème
UPnP permet à n'importe quel appareil d'ouvrir des ports vers Internet sans intervention humaine. Trois ports avaient été ouverts automatiquement (554, 5353, 8080).

### Actions

**Désactiver UPnP depuis l'interface d'administration de la box**

Accès : `http://192.168.1.1` → Réseau / Pare-feu → UPnP → Désactiver

Après désactivation, vérification :

```bash
upnpc -l
```

```
Found valid IGD : http://192.168.1.1:1900/desc/root
No port mapping found.
```

**Vérification des ports ouverts depuis Internet**

On utilise un scan depuis l'extérieur pour confirmer qu'aucun port n'est plus exposé. Via le service en ligne `nmap.online` ou depuis un VPS externe :

```bash
# Depuis un serveur externe — IP publique de la box
nmap -p 554,8080,5353 [IP_PUBLIQUE_ANONYMISÉE]
```

```
PORT     STATE    SERVICE
554/tcp  filtered rtsp
5353/tcp filtered mdns
8080/tcp filtered http-alt
```

`filtered` = le trafic est bloqué. Les ports ne sont plus accessibles depuis Internet.

---

## Remédiation 3 — PC Windows : désactivation SMB inutile

### Problème
SMB actif avec ports 135, 139, 445 ouverts. IPC$ accessible en guest. Le partage de fichiers n'est pas utilisé sur ce poste.

### Actions

**Via PowerShell (avec droits administrateur) :**

```powershell
# Désactiver le service SMBv1 (obsolète et dangereux)
Set-SmbServerConfiguration -EnableSMB1Protocol $false

# Désactiver le partage de fichiers si non nécessaire
Set-SmbServerConfiguration -EnableSMB2Protocol $false

# Désactiver NetBIOS over TCP/IP
# Via Panneau de configuration > Réseau > Propriétés IPv4 > Avancé > WINS > Désactiver NetBIOS
```

**Règles de pare-feu Windows :**

```powershell
# Bloquer SMB entrant depuis le réseau local
netsh advfirewall firewall add rule name="Block SMB Inbound" dir=in action=block protocol=tcp localport=139,445
```

**Vérification post-remédiation :**

```bash
# Depuis Kali — rescan du PC Windows
sudo nmap -p 135,139,445 192.168.1.102
```

```
PORT    STATE    SERVICE
135/tcp filtered msrpc
139/tcp filtered netbios-ssn
445/tcp filtered microsoft-ds
```

Ports filtrés. SMB n'est plus accessible depuis le réseau.

---

## Remédiation 4 — NAS Synology : restreindre l'accès externe

### Problème
Le port 8080 (interface admin Synology) avait été exposé vers Internet via UPnP.

### Actions

UPnP désactivé = la règle a disparu automatiquement. On vérifie néanmoins la configuration du NAS :

- Désactiver l'accès QuickConnect si non utilisé (Panneau de configuration → QuickConnect)
- Activer le pare-feu intégré DSM : Panneau de configuration → Sécurité → Pare-feu → Activer
- Règle : bloquer tout trafic entrant sauf depuis 192.168.1.0/24

---

## Mise en place de règles iptables sur la machine Linux

Pour durcir notre propre machine Kali utilisée comme poste d'audit, on configure un pare-feu local minimal via `iptables` :

```bash
# Politique par défaut : tout bloquer en entrée
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Autoriser les connexions établies et relatives
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Autoriser le loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# Autoriser SSH uniquement depuis le réseau local
sudo iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT

# Sauvegarder les règles
sudo iptables-save > /etc/iptables/rules.v4
```

**Vérification :**

```bash
sudo iptables -L -n -v
```

```
Chain INPUT (policy DROP)
target  prot  opt  source           destination
ACCEPT  all   --   anywhere         anywhere     state RELATED,ESTABLISHED
ACCEPT  all   --   anywhere         anywhere     (loopback)
ACCEPT  tcp   --   192.168.1.0/24   anywhere     tcp dpt:22
```

---

## Bilan final : avant / après

| Vulnérabilité | Avant | Après |
|---|---|---|
| Caméra — identifiants par défaut | admin:admin | Mot de passe fort appliqué |
| Caméra — CVE-2017-8225 | Firmware 2018, non patché | Firmware mis à jour |
| Caméra — RTSP exposé Internet | Port 554 ouvert publiquement | Port fermé, UPnP désactivé |
| Caméra — isolation réseau | Sur le LAN principal | Isolée en VLAN IoT |
| Gateway — UPnP | Actif, 3 ports ouverts | Désactivé |
| PC Windows — SMB | Actif, IPC$ en guest | Désactivé |
| NAS — admin exposé | Port 8080 ouvert Internet | Fermé, pare-feu DSM activé |
| Machine Linux — pare-feu | Aucune règle | iptables configuré |

---

## Conclusion du projet

Ce qui semblait être un réseau banal — une caméra, une box, quelques appareils — s'est révélé être un périmètre avec plusieurs points d'entrée exploitables, dont un directement accessible depuis Internet avec un exploit public disponible.

L'audit n'a pas créé les vulnérabilités. Il les a rendues visibles.

C'est ça, la valeur de la démarche : transformer un réseau opaque en une infrastructure que l'on comprend, que l'on surveille, et que l'on peut défendre.

---

*Précédent : [P3 — Évaluation des Vulnérabilités](../P3-Vulnerabilites/README.md)*
*Retour à l'index : [README principal](../README.md)*
