# Network Insight & Security Analysis

> *"Un matin, je me suis posé une question simple : est-ce que je sais vraiment ce qui transite sur mon propre réseau ?"*

---

## Pourquoi ce projet ?

La plupart des gens font confiance à leur réseau domestique sans jamais le regarder vraiment. Une box, un mot de passe WiFi, quelques appareils connectés — et on suppose que tout va bien.

Mais supposer, en sécurité, c'est la première erreur.

Ce projet est né d'une curiosité simple qui est devenue une démarche structurée : **auditer mon propre réseau comme si c'était un périmètre professionnel**. Pas juste scanner des ports pour le fun. Comprendre ce qui est là, ce qui parle, à qui, et pourquoi. Détecter ce qui ne devrait pas être là. Et corriger.

Ce dépôt documente cette démarche en quatre phases, de la reconnaissance initiale jusqu'au durcissement du périmètre.

---

## Note sur l'anonymisation

> ⚠️ **Toutes les données réseau présentées dans ce projet ont été anonymisées.**

Dans tout audit de sécurité professionnel, on ne publie jamais les vraies adresses IP, adresses MAC, noms de réseaux (SSID) ou informations d'identification d'un système audité. Ce n'est pas une limite — c'est une règle éthique fondamentale.

Les valeurs présentes dans ce dépôt (adresses IP, MAC, noms d'hôtes) sont **fictives mais réalistes**. Elles suivent les mêmes formats, les mêmes plages, les mêmes structures que ce qu'on trouverait sur un vrai réseau domestique. L'objectif est de comprendre la démarche, pas d'exposer une infrastructure réelle.

---

## Environnement de travail

| Élément | Détail |
|---|---|
| Système | Kali Linux |
| Interface réseau | `eth0` / `wlan0` |
| Outils principaux | `nmap`, `arp-scan`, `netdiscover`, `tcpdump`, `Wireshark`, `searchsploit` |
| Périmètre audité | Réseau domestique — traité comme un réseau d'entreprise |

---

## Structure du projet

| Phase | Désignation | État |
|---|---|---|
| [P1 — Reconnaissance et Inventaire](./P1-Reconnaissance/README.md) | Cartographier ce qui existe | ✅ Terminée |
| [P2 — Analyse de Flux et Trafic](./P2-Analyse-Trafic/README.md) | Comprendre ce qui circule | ✅ Terminée |
| [P3 — Évaluation des Vulnérabilités](./P3-Vulnerabilites/README.md) | Identifier ce qui est exposé | ✅ Terminée |
| [P4 — Remédiation et Hardening](./P4-Remediation/README.md) | Corriger et durcir | ✅ Terminée |

---

## Ce que ce projet m'a appris

Un réseau "calme" n'est pas un réseau sûr. C'est juste un réseau qu'on ne regarde pas.

La visibilité précède tout. On ne peut pas défendre ce qu'on ne voit pas, et on ne peut pas voir ce qu'on n'a pas inventorié. Cette démarche m'a montré, concrètement, comment passer d'un réseau opaque à un périmètre observé, analysé et durci.

---

*Projet réalisé dans le cadre d'une démarche personnelle d'apprentissage en cybersécurité.*
