# Network Insight & Security Analysis
> Audit de sécurité et monitoring d'une infrastructure réseau hybride.

---

## Introduction
"Un matin, je me suis posé une question centrale : quelle est la visibilité réelle sur les flux transitant par mon réseau ?"

La visibilité est le premier rempart de la défense. Ce projet documente ma démarche d'analyste pour transformer un environnement réseau domestique opaque en une infrastructure surveillée et segmentée, capable de résister aux vecteurs d'attaque modernes.

---

## Note de Cadrage

### 1.1 Contexte du Projet
L'audit porte sur un périmètre réseau domestique. La problématique identifiée est l'opacité des flux : de nombreux terminaux (mobiles, IoT) communiquent avec l'extérieur sans contrôle, créant des risques potentiels d'exfiltration de données ou d'intrusion.

### 1.2 Objectifs Stratégiques
* **Observabilité** : Identification de 100% des actifs et cartographie des interactions.
* **Segmentation** : Application du principe du moindre privilège par l'isolation des zones à risque.
* **Intégrité** : Détection des communications anormales et des tentatives de connexion non autorisées.

---

## Bagage de savoir minimal
Avant de débuter l'audit technique, voici les piliers conceptuels du projet :

1. **La Visibilité** : On ne protège que ce que l'on voit. L'objectif est de lever le voile sur les actifs dits "invisibles".
2. **Le Flux de données** : L'analyse des protocoles permet de valider le comportement nominal d'un équipement.
3. **La Segmentation** : Création de barrières virtuelles pour empêcher la propagation d'une menace (mouvement latéral).

---

## Glossaire Technique

### Sécurité & Concepts
* **Audit** : Inspection rigoureuse visant à mesurer la conformité d'un système.
* **Zero Trust** : Modèle de sécurité où aucun appareil n'est considéré comme fiable par défaut.
* **Surface d'attaque** : Somme des points d'entrée vulnérables d'un système.
* **Hardening** : Processus de sécurisation d'un système par la réduction de sa surface d'exposition.
* **CVE** : Base de données répertoriant les vulnérabilités de sécurité connues.

### Réseau & Analyse
* **Actifs (Assets)** : Ensemble des équipements physiques ou virtuels connectés.
* **Fingerprinting** : Identification d'un système d'exploitation par l'analyse de ses réponses réseau.
* **Sniffing** : Capture et analyse de paquets de données en temps réel.
* **DNS** : Protocole de résolution de noms de domaine en adresses IP.
* **VLAN** : Segmentation logique d'un réseau physique.

### Menaces
* **Shadow IT** : Utilisation de systèmes ou d'appareils non approuvés par l'administrateur.
* **Mouvement Latéral** : Technique consistant à s'étendre dans un réseau après un premier point d'entrée.
* **C2 (Command & Control)** : Infrastructure utilisée par un attaquant pour piloter un système compromis.

---

## Méthodologie et État d'avancement
| Phase | Désignation | État |
| :--- | :--- | :--- |
| **P1** | Reconnaissance et Inventaire | En cours |
| **P2** | Analyse de Flux et Trafic | À venir |
| **P3** | Évaluation des Vulnérabilités | À venir |
| **P4** | Remédiation et Hardening | À venir |
C2 (Command & Control) : Un serveur externe utilisé par un pirate pour diriger à distance des appareils infectés sur ton réseau.



