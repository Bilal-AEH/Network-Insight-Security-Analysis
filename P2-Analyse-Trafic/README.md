# P2 — Analyse de Flux et Trafic

> *Savoir qu'un appareil est là, c'est bien. Savoir ce qu'il dit — et à qui — c'est autre chose.*

---

## Objectif de cette phase

L'inventaire de P1 nous a montré **qui** est sur le réseau. Cette phase répond à une question différente : **que font ces appareils, exactement ?**

Un réseau n'est pas statique. Les appareils parlent en permanence — parfois de façon attendue, parfois de façon surprenante. Une caméra qui envoie du trafic vers un serveur chinois à 3h du matin, un téléphone qui interroge un domaine inconnu, un appareil qui scanne silencieusement ses voisins — tout ça se voit dans le trafic, si on sait regarder.

C'est l'objet de cette phase : **capturer, analyser, interpréter**.

---

## Outils utilisés

| Outil | Rôle |
|---|---|
| `tcpdump` | Capture de paquets en ligne de commande |
| `Wireshark` | Analyse graphique approfondie des captures |
| `tshark` | Version CLI de Wireshark — filtres précis, traitement en masse |

---

## Étape 1 — Mise en place de la capture

Avant de capturer, une question se pose : sur quel trafic ? On est sur un réseau switché — contrairement à un réseau hub, chaque appareil ne voit normalement que ce qui lui est destiné. Pour voir le trafic des autres, deux approches :

- **Capture sur la gateway** (si on y a accès)
- **ARP Poisoning / MITM** (technique offensive — hors périmètre de cette phase)
- **Capture de son propre trafic sortant** + analyse du broadcast

On commence par ce qu'on peut faire sans manipuler le réseau : écouter le trafic broadcast et le trafic propre à notre interface.

```bash
sudo tcpdump -i eth0 -w capture_p2.pcap
```

> `-i eth0` : interface d'écoute
> `-w capture_p2.pcap` : enregistrement dans un fichier au format PCAP (Packet Capture) pour analyse ultérieure dans Wireshark

La capture tourne pendant **15 minutes** — assez pour observer les cycles de communication des appareils.

```bash
# Arrêt propre après 15 min
sudo tcpdump -i eth0 -G 900 -W 1 -w capture_p2.pcap
```

---

## Étape 2 — Analyse avec tshark : vue d'ensemble

On commence par une vue macro : quels sont les protocoles présents, quels hôtes communiquent le plus ?

```bash
tshark -r capture_p2.pcap -q -z io,phs
```

**Résultat (extrait) :**

```
===================================================================
Protocol Hierarchy Statistics
Filter: 

eth                                      frames:8432 bytes:1243891
  ip                                     frames:7201 bytes:1198432
    tcp                                  frames:4832 bytes:987234
      tls                                frames:3901 bytes:912043
      http                               frames:218  bytes:41233
    udp                                  frames:2369 bytes:211198
      dns                                frames:892  bytes:87432
      mdns                               frames:341  bytes:32109
      ssdp                               frames:127  bytes:11203
  arp                                    frames:1231 bytes:45459
```

**Lecture :**
- La majorité du trafic TCP est chiffré (TLS) — c'est normal pour du HTTPS
- 892 requêtes DNS en 15 minutes — c'est beaucoup. À analyser
- MDNS et SSDP présents — protocoles de découverte automatique des appareils sur le réseau local

> 💡 **MDNS** (Multicast DNS) : permet aux appareils de se retrouver sur un réseau local sans serveur DNS central. Utilisé par AirPlay, Chromecast, imprimantes, etc.

> 💡 **SSDP** (Simple Service Discovery Protocol) : composant d'UPnP. Permet la découverte automatique de services. Souvent exploité pour des attaques par amplification DDoS.

---

## Étape 3 — Analyse DNS : que cherchent les appareils ?

892 requêtes DNS en 15 minutes. C'est le premier signal intéressant. Le DNS est le "bottin téléphonique" d'Internet — avant de se connecter à un serveur, un appareil demande son adresse IP. En regardant les requêtes DNS, on voit les destinations avant même d'analyser les connexions.

```bash
tshark -r capture_p2.pcap -Y "dns.flags.response == 0" -T fields \
  -e ip.src -e dns.qry.name | sort | uniq -c | sort -rn | head 20
```

**Résultat (anonymisé) :**

```
 187  192.168.1.87   alexa.amazon.com
 143  192.168.1.87   api.amazonalexa.com
  89  192.168.1.34   p2p.lc-cloud-service.net
  67  192.168.1.34   cdn.lc-ipc-service.com
  54  192.168.1.102  time.windows.com
  43  192.168.1.102  settings-win.data.microsoft.com
  38  192.168.1.120  clients3.google.com
  21  192.168.1.21   api.apple-cloudkit.com
  12  192.168.1.10   github.com
   8  192.168.1.15   nas.local
```

**Ce qu'on observe :**

L'Amazon Echo (192.168.1.87) communique massivement avec les serveurs Amazon — attendu. Le Chromecast avec Google — attendu. L'iPhone avec Apple — attendu.

Ce qui accroche : **la caméra IP (192.168.1.34)** interroge des domaines comme `p2p.lc-cloud-service.net` et `cdn.lc-ipc-service.com`. Ces domaines ne correspondent à aucun fabricant connu — ils évoquent un service cloud tiers non documenté. C'est un signal à creuser.

---

## Étape 4 — Filtrage par hôte suspect : la caméra IP

On isole tout le trafic de la caméra pour comprendre ses connexions.

```bash
tshark -r capture_p2.pcap -Y "ip.src == 192.168.1.34 or ip.dst == 192.168.1.34" \
  -T fields -e frame.time -e ip.src -e ip.dst -e ip.proto -e tcp.dstport
```

**Extrait du résultat :**

```
2025-11-14 02:17:43  192.168.1.34  45.32.187.x  6   443
2025-11-14 02:17:45  192.168.1.34  103.224.182.x 6  8080
2025-11-14 02:18:01  192.168.1.34  45.32.187.x  6   443
2025-11-14 02:31:17  192.168.1.34  103.224.182.x 6  8080
2025-11-14 02:31:19  192.168.1.34  45.32.187.x  6   443
```

**Ce qu'on observe :**
- La caméra établit des connexions sortantes toutes les ~14 minutes
- Elle contacte deux IPs distinctes sur les ports 443 et **8080**
- Le port 8080 sur une connexion vers un serveur distant est inhabituel — 443 (HTTPS) serait standard pour une mise à jour ou un heartbeat cloud

Une géolocalisation rapide des IPs distantes (via `whois`) indique des serveurs localisés en **Asie du Sud-Est** — cohérent avec un firmware de caméra IP de marque générique.

> 💡 **Heartbeat** : signal périodique envoyé par un équipement pour signaler qu'il est actif. Sur un appareil IoT, ça peut aussi servir à la télémétrie ou au contrôle à distance.

---

## Étape 5 — Analyse du trafic broadcast

Le trafic broadcast (envoyé à tous les appareils du réseau) est particulièrement informatif. Il révèle les protocoles de découverte actifs et parfois des comportements inattendus.

```bash
tshark -r capture_p2.pcap -Y "eth.dst == ff:ff:ff:ff:ff:ff" \
  -T fields -e ip.src -e udp.dstport -e _ws.col.Protocol | sort | uniq -c
```

**Résultat :**

```
 127  192.168.1.1    1900  SSDP
  89  192.168.1.87   1900  SSDP
  67  192.168.1.102  137   NBNS
  41  192.168.1.15   1900  SSDP
  23  192.168.1.34   5353  MDNS
```

> 💡 **NBNS** (NetBIOS Name Service) : protocole Windows de résolution de noms sur le réseau local. Sa présence sur le port 137 confirme que le PC Windows utilise encore NetBIOS — une vieille technologie avec un historique de vulnérabilités connu.

---

## Bilan de la phase 2

| Observation | Source | Risque |
|---|---|---|
| Connexions sortantes régulières vers serveurs asiatiques | Caméra IP (192.168.1.34) | 🔴 Élevé |
| SSDP actif sur plusieurs appareils | Gateway, Echo, NAS | 🟡 Moyen |
| NetBIOS actif sur le PC Windows | 192.168.1.102 | 🟡 Moyen |
| Volume DNS élevé de l'Echo | 192.168.1.87 | 🟢 Normal (Amazon) |
| Trafic TLS majoritaire | Global | 🟢 Normal |

La caméra IP concentre l'essentiel des anomalies observées. Elle sera la cible prioritaire de la phase d'évaluation des vulnérabilités.

---

*Précédent : [P1 — Reconnaissance et Inventaire](../P1-Reconnaissance/README.md)*
*Suite : [P3 — Évaluation des Vulnérabilités](../P3-Vulnerabilites/README.md)*
