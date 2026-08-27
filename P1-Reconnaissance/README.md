# P1 — Reconnaissance et Inventaire

> *Avant de défendre, il faut voir. Avant de voir, il faut chercher.*

---

## Objectif de cette phase

La première question à se poser face à un réseau n'est pas "est-il sécurisé ?". C'est : **"Qu'est-ce qui s'y trouve ?"**

Sans inventaire, tout le reste est aveugle. On ne peut pas détecter une anomalie si on ne sait pas ce qui est normal. On ne peut pas bloquer un accès non autorisé si on ne connaît pas les accès légitimes.

Cette phase a un seul but : **cartographier l'intégralité des actifs (assets) présents sur le réseau**, avec leur adresse IP, leur adresse MAC, leur nom d'hôte si disponible, et leur système d'exploitation si identifiable.

---

## Méthodologie

La reconnaissance s'est déroulée en trois temps :

1. **Découverte passive** — écouter ce qui se passe sans envoyer de paquets
2. **Découverte active légère** — envoyer des requêtes ARP pour recenser les actifs
3. **Scan approfondi** — utiliser Nmap pour fingerprinter les systèmes

---

## Étape 1 — Identifier son propre contexte réseau

Avant de scanner quoi que ce soit, il faut connaître sa propre position dans le réseau. Sur quelle interface est-on ? Quelle est notre adresse IP ? Quelle est la plage du réseau ?

```bash
ip a
ip route
```

**Ce qu'on observe :**

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.1.42/24 brd 192.168.1.255 scope global eth0

default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link
```

**Ce qu'on en tire :**
- Notre machine est à `192.168.1.42`
- La passerelle (gateway) — c'est-à-dire le routeur/box — est à `192.168.1.1`
- La plage réseau à scanner est `192.168.1.0/24` — soit jusqu'à 254 hôtes possibles

> 💡 **Gateway** (passerelle) : c'est l'équipement qui fait le lien entre notre réseau local et Internet. Sur un réseau domestique, c'est généralement la box opérateur.

---

## Étape 2 — Découverte des actifs avec ARP-Scan

Le protocole **ARP** (Address Resolution Protocol) est utilisé par les équipements réseau pour associer une adresse IP à une adresse MAC. En envoyant des requêtes ARP sur toute la plage réseau, on obtient la liste de tous les appareils actifs — sans déclencher d'alerte, car c'est un trafic tout à fait normal sur un réseau local.

```bash
sudo arp-scan --localnet
```

**Résultat obtenu (anonymisé) :**

```
Interface: eth0, type: EN10MB, MAC: aa:bb:cc:11:22:33, IPv4: 192.168.1.42
Starting arp-scan 1.10.0 with 256 hosts

192.168.1.1     d4:6e:0e:xx:xx:xx    (Routeur — Box opérateur)
192.168.1.10    b8:27:eb:xx:xx:xx    (Raspberry Pi Foundation)
192.168.1.15    00:11:32:xx:xx:xx    (Synology Inc. — NAS)
192.168.1.21    f0:18:98:xx:xx:xx    (Apple — iPhone)
192.168.1.34    dc:a6:32:xx:xx:xx    (Raspberry Pi — caméra IP)
192.168.1.87    50:3e:aa:xx:xx:xx    (Amazon — Echo Dot)
192.168.1.102   00:e0:4c:xx:xx:xx    (Realtek — PC Windows)
192.168.1.120   18:31:bf:xx:xx:xx    (Google — Chromecast)

8 hosts found, 0 duplicates, 0 errors
```

**Ce qu'on observe :**
8 actifs identifiés. Le préfixe OUI (les 3 premiers octets de l'adresse MAC) permet d'identifier le fabricant de la carte réseau — ce qui donne souvent un indice sur le type d'appareil.

> 💡 **Adresse MAC** (Media Access Control) : identifiant unique attribué à chaque interface réseau. Les 3 premiers octets identifient le fabricant (OUI — Organizationally Unique Identifier). C'est permanent et gravé dans le matériel.

---

## Étape 3 — Scan réseau avec Nmap

`arp-scan` nous donne la liste. `nmap` va plus loin : il analyse les **ports ouverts**, tente d'identifier le **système d'exploitation** et les **services** qui tournent sur chaque machine.

### Scan de découverte rapide

```bash
sudo nmap -sn 192.168.1.0/24
```

> `-sn` : ping scan — on ne scanne pas les ports, on vérifie juste qui est vivant. Rapide, peu intrusif.

```
Starting Nmap 7.94 ( https://nmap.org )
Nmap scan report for 192.168.1.1
Host is up (0.0032s latency).
MAC Address: D4:6E:0E:XX:XX:XX (Routeur opérateur)

Nmap scan report for 192.168.1.10
Host is up (0.0018s latency).
MAC Address: B8:27:EB:XX:XX:XX (Raspberry Pi Foundation)

[... 6 autres hôtes ...]

Nmap done: 256 IP addresses (8 hosts up) scanned in 3.42 seconds
```

### Scan approfondi sur les actifs identifiés

Une fois les hôtes listés, on lance un scan plus précis sur chacun — ports ouverts, services, versions, OS.

```bash
sudo nmap -sS -sV -O 192.168.1.1
```

> `-sS` : SYN scan (discret, n'établit pas la connexion complète)
> `-sV` : détection de version des services
> `-O` : fingerprinting du système d'exploitation

**Résultat sur la gateway (192.168.1.1) :**

```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        Dropbear sshd 2020.81
53/tcp   open  domain     dnsmasq 2.86
80/tcp   open  http       mini_httpd 1.30
443/tcp  open  ssl/https  mini_httpd 1.30
1900/udp open  upnp       MiniUPnPd 2.1

OS detection: Linux 3.x (embedded)
```

**Ce qu'on observe et ce que ça signifie :**

| Port | Service | Observation |
|---|---|---|
| 22 | SSH | Interface d'administration à distance — à surveiller |
| 53 | DNS | Le routeur fait office de résolveur DNS local |
| 80/443 | HTTP/HTTPS | Interface web d'administration de la box |
| 1900 | UPnP | Protocole de découverte automatique — vecteur d'attaque connu |

> 💡 **UPnP** (Universal Plug and Play) : protocole qui permet aux appareils de s'auto-configurer sur un réseau. Pratique, mais régulièrement exploité par des malwares pour ouvrir des ports sans intervention humaine.

---

## Étape 4 — Scan des autres actifs

Même procédure appliquée aux autres hôtes identifiés. Exemple sur le Raspberry Pi (192.168.1.10) :

```bash
sudo nmap -sS -sV -O 192.168.1.10
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2 (protocol 2.0)
80/tcp open  http    nginx 1.22.1

OS detection: Linux 5.15 (Raspberry Pi OS)
```

Le Raspberry Pi expose un serveur web en local. C'est attendu si c'est un serveur de dashbord ou de domotique — mais ça mérite d'être noté et vérifié.

---

## Récapitulatif des actifs identifiés

| IP | Type d'appareil | OS estimé | Ports ouverts | Niveau de risque |
|---|---|---|---|---|
| 192.168.1.1 | Routeur / Box | Linux embarqué | 22, 53, 80, 443, 1900 | 🟡 Moyen (UPnP actif) |
| 192.168.1.10 | Raspberry Pi | Raspberry Pi OS | 22, 80 | 🟢 Faible |
| 192.168.1.15 | NAS Synology | Linux (DSM) | 80, 443, 5000, 5001 | 🟡 Moyen |
| 192.168.1.21 | iPhone | iOS | Aucun port ouvert | 🟢 Faible |
| 192.168.1.34 | Caméra IP | Linux embarqué | 80, 554 (RTSP) | 🔴 Élevé |
| 192.168.1.87 | Amazon Echo | Linux (Amazon) | 4070, 55442 | 🟡 Moyen |
| 192.168.1.102 | PC Windows | Windows 10/11 | 135, 139, 445 | 🟡 Moyen (SMB) |
| 192.168.1.120 | Chromecast | Chrome OS (lite) | 8008, 8009 | 🟢 Faible |

> 💡 **SMB** (Server Message Block) : protocole Windows de partage de fichiers et d'imprimantes. Les ports 139 et 445 ouverts signalent un partage actif — vecteur historique de nombreux ransomwares (WannaCry, NotPetya).

> 💡 **RTSP** (Real Time Streaming Protocol) : protocole de streaming vidéo. Port 554 ouvert sur une caméra IP sans authentification robuste = accès potentiel au flux vidéo depuis le réseau local.

---

## Ce que cette phase révèle

Trois observations importantes à retenir avant de passer à l'analyse de trafic :

**1. La caméra IP est le point le plus exposé.** Port RTSP ouvert, système embarqué probablement non mis à jour — c'est le profil classique d'un appareil IoT vulnérable.

**2. UPnP est actif sur la gateway.** Ce protocole permet à n'importe quel appareil du réseau d'ouvrir des ports sur la box sans demander de permission. C'est une surface d'attaque à surveiller.

**3. Le PC Windows expose SMB.** Si le partage de fichiers n'est pas nécessaire, ces ports n'ont pas à être ouverts. Ils seront vérifiés en phase 3.

---

*Suite : [P2 — Analyse de Flux et Trafic](../P2-Analyse-Trafic/README.md)*
