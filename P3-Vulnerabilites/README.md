# P3 — Évaluation des Vulnérabilités

> *Identifier ce qui existe, c'est la phase 1. Comprendre ce qui est cassé, c'est ici.*

---

## Objectif de cette phase

Les deux phases précédentes ont dessiné une carte : on sait qui est là (P1) et ce que les appareils font (P2). Maintenant, on pose la question qui dérange : **est-ce que ces appareils sont vulnérables ?**

Un port ouvert n'est pas un problème en soi. Un service avec une version non patchée derrière ce port, oui. Cette phase consiste à croiser les services découverts avec les bases de données de vulnérabilités connues, et à évaluer le niveau de risque réel.

On ne cherche pas à exploiter quoi que ce soit. On cherche à **mesurer l'exposition**.

---

## Cibles prioritaires

D'après l'analyse P1 et P2, trois actifs concentrent le risque :

1. **Caméra IP (192.168.1.34)** — firmware embarqué, connexions vers des serveurs non documentés
2. **PC Windows (192.168.1.102)** — ports SMB ouverts, NetBIOS actif
3. **Gateway/Box (192.168.1.1)** — UPnP actif, interface web exposée

---

## Étape 1 — Scan de vulnérabilités avec Nmap NSE

Nmap embarque un moteur de scripts appelé **NSE** (Nmap Scripting Engine). Ces scripts permettent d'aller plus loin qu'un simple scan de ports — ils testent des vulnérabilités spécifiques, vérifient des configurations, récupèrent des informations de version précises.

```bash
sudo nmap -sV --script=vuln 192.168.1.34
```

> `--script=vuln` : charge tous les scripts de la catégorie "vulnérabilités" du NSE

**Résultat sur la caméra IP :**

```
Starting Nmap 7.94
Nmap scan report for 192.168.1.34

PORT   STATE SERVICE VERSION
80/tcp open  http    Boa httpd 0.94.14rc21
|_http-server-header: Boa/0.94.14rc21
| http-vuln-cve2017-8225:
|   VULNERABLE:
|   CVE-2017-8225 — GoAhead Embedded Web Server Path Traversal
|     State: VULNERABLE
|     Risk factor: HIGH
|     Description: Allows unauthenticated remote access to sensitive files
|     References: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-8225

554/tcp open  rtsp    H264DVR rtsp
|_rtsp-methods: OPTIONS, DESCRIBE, SETUP, PLAY, TEARDOWN
| rtsp-url-brute:
|   Found credentials: admin:admin
|   Stream URL: rtsp://192.168.1.34:554/user=admin&password=admin&channel=1&stream=0
```

**Ce qu'on observe — et c'est sérieux :**

Deux problèmes critiques sur cette caméra :

- **CVE-2017-8225** : une vulnérabilité connue depuis 2017 permettant à n'importe qui sur le réseau local de lire des fichiers sensibles (configuration, mots de passe) sans s'authentifier. La caméra n'a pas reçu de mise à jour firmware depuis plusieurs années.

- **Identifiants par défaut non changés** : `admin:admin`. Le flux RTSP est accessible à quiconque connaît l'IP. Si la box expose le port 554 vers Internet (ce qu'UPnP peut avoir fait automatiquement), c'est accessible depuis n'importe où dans le monde.

> 💡 **CVE** (Common Vulnerabilities and Exposures) : système de référencement international des vulnérabilités de sécurité. Chaque CVE a un identifiant unique (CVE-ANNÉE-NUMÉRO) et un score de sévérité (CVSS).

---

## Étape 2 — Recherche dans la base Exploit-DB

`searchsploit` est un outil qui permet d'interroger la base **Exploit-DB** — une archive d'exploits publics correspondant à des CVE connues. Ici, on cherche des exploits existants pour le firmware de la caméra.

```bash
searchsploit "Boa httpd"
searchsploit "H264DVR"
```

**Résultats :**

```
Exploit-DB | Title                                              | Path
-----------|----------------------------------------------------|-----------------------
EDB-41471  | Boa HTTPd 0.94 - Remote Denial of Service          | exploits/linux/dos/
EDB-43228  | H264DVR - Remote Code Execution (Metasploit)       | exploits/hardware/remote/
EDB-44097  | IP Camera - Configuration Disclosure (Unauthenticated) | exploits/hardware/webapps/
```

Un module **Metasploit** existe pour l'exécution de code à distance sur ce type de DVR/caméra. Ce n'est pas une démonstration théorique — c'est un exploit packagé, utilisable en quelques commandes.

> 💡 **Metasploit** : framework open-source utilisé en pentest pour développer et exécuter des exploits. Utilisé légalement pour tester la sécurité de ses propres systèmes.

---

## Étape 3 — Évaluation du PC Windows (SMB)

```bash
sudo nmap -sV --script=smb-vuln-ms17-010 192.168.1.102
```

> `smb-vuln-ms17-010` : teste la vulnérabilité EternalBlue, utilisée par WannaCry en 2017

**Résultat :**

```
PORT    STATE SERVICE       VERSION
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds  Windows 10 microsoft-ds

Host script results:
| smb-vuln-ms17-010:
|   NOT VULNERABLE
|   EternalBlue check succeeded — patch applied.
```

Bonne nouvelle : le PC est à jour sur EternalBlue. En revanche, SMB reste actif et expose deux ports sur le réseau local. On vérifie si un partage est accessible sans authentification :

```bash
sudo nmap --script=smb-enum-shares 192.168.1.102
```

```
| smb-enum-shares:
|   account_used: guest
|   \\192.168.1.102\ADMIN$:
|     Type: STYPE_DISKTREE_HIDDEN
|     Access: NO ACCESS
|   \\192.168.1.102\C$:
|     Type: STYPE_DISKTREE_HIDDEN
|     Access: NO ACCESS
|   \\192.168.1.102\IPC$:
|     Type: STYPE_IPC_HIDDEN
|     Access: READ ONLY (via guest)
```

IPC$ est accessible en lecture par un compte invité. Ce n'est pas critique en soi, mais c'est une surface d'attaque qui n'est pas nécessaire si le partage de fichiers n'est pas utilisé.

---

## Étape 4 — Vérification UPnP sur la gateway

L'UPnP actif sur la box peut avoir ouvert des ports vers Internet automatiquement. On vérifie quels ports ont été ouverts :

```bash
upnpc -l
```

**Résultat :**

```
Found valid IGD : http://192.168.1.1:1900/desc/root
Local LAN ip address : 192.168.1.42

 #  Protocol  ExPort->InAddr:InPort  Description  Enabled  RemoteHost  LeaseTime
 0  TCP        554->192.168.1.34:554  H264DVR       Yes      ''          0
 1  UDP       5353->192.168.1.87:5353 AmazonEcho    Yes      ''          0
 2  TCP       8080->192.168.1.15:5000 Synology NAS  Yes      ''          0
```

**C'est là que tout se connecte.**

UPnP a ouvert automatiquement le port 554 (le flux RTSP de la caméra) vers Internet. Combiné aux identifiants `admin:admin` et à la CVE-2017-8225, la caméra est **accessible depuis n'importe où dans le monde** et **exploitable sans authentification**.

---

## Récapitulatif des vulnérabilités identifiées

| Actif | Vulnérabilité | CVE / Référence | Score CVSS | Priorité |
|---|---|---|---|---|
| Caméra IP | Identifiants par défaut (admin:admin) | — | 9.8 | 🔴 Critique |
| Caméra IP | Path Traversal firmware non patché | CVE-2017-8225 | 9.8 | 🔴 Critique |
| Caméra IP | Port RTSP exposé vers Internet via UPnP | EDB-43228 | 9.0 | 🔴 Critique |
| Gateway | UPnP actif — ouverture automatique de ports | — | 7.5 | 🟡 Élevé |
| NAS Synology | Interface admin exposée vers Internet (port 8080) | — | 7.2 | 🟡 Élevé |
| PC Windows | SMB actif inutilement, IPC$ en accès guest | — | 4.3 | 🟡 Moyen |

---

## Ce que cette phase confirme

On part d'un réseau "normal" — une caméra, une box, un PC. Et on se retrouve avec une caméra entièrement compromise potentiellement depuis Internet, avec son flux vidéo accessible publiquement et un exploit Metasploit disponible pour du RCE (Remote Code Execution — exécution de code à distance).

Ce n'est pas un scénario théorique. C'est ce qu'on trouve sur des milliers de réseaux domestiques.

> 💡 **RCE** (Remote Code Execution) : vulnérabilité permettant à un attaquant d'exécuter du code arbitraire sur la machine cible à distance. C'est la classe de vulnérabilité la plus critique qui existe.

---

*Précédent : [P2 — Analyse de Flux et Trafic](../P2-Analyse-Trafic/README.md)*
*Suite : [P4 — Remédiation et Hardening](../P4-Remediation/README.md)*
