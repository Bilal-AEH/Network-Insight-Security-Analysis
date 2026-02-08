# Network-Insight-Security-Analysis
"Un matin, je me suis réveillé avec une question simple : Est-ce que je sais vraiment ce qui transite sur mon propre réseau ?"

0. Avant-propos & Accroche
   "La visibilité est la première étape de la défense. On ne peut pas protéger ce que l'on ne voit pas."

Ce projet est né d'un constat simple : la multiplication des équipements connectés (IoT, terminaux mobiles, domotique) transforme chaque foyer en une infrastructure complexe, souvent dépourvue de supervision. Ce dépôt documente ma démarche d'analyste réseau pour reprendre le contrôle total sur les flux de données et sécuriser mon environnement contre les menaces modernes.

1. Note de Cadrage
   1.1 Contexte du Projet
L'audit porte sur un périmètre réseau domestique hybride. La problématique centrale est l'opacité des flux : de nombreux périphériques communiquent vers l'extérieur sans contrôle préalable, créant des vecteurs d'exfiltration de données ou d'intrusion potentiels.

 1.2 Objectifs Stratégiques
Le projet vise à répondre à trois piliers de la sécurité informatique :

Observabilité : Identifier 100% des actifs connectés et cartographier leurs interactions.

Segmentation : Appliquer le principe du moindre privilège en isolant les équipements à risque.

Intégrité : Détecter toute communication anormale ou tentative de connexion non autorisée.


Avant d'entrer dans le vif de l'audit, il est essentiel de comprendre quelques piliers qui soutiennent ce projet. Voici les concepts clés à maîtriser pour suivre mon cheminement :

1. La visibilité (On ne protège que ce que l'on voit)
Le plus gros risque dans un réseau n'est pas forcément une attaque complexe, mais l'existence d'appareils "invisibles" ou oubliés. Mon premier travail est de lever le voile sur tout ce qui est branché.

2. Le flux de données (Le réseau est vivant)
Un appareil ne se contente pas d'être "connecté" ; il discute sans arrêt. Comprendre ces discussions (les protocoles), c'est être capable de dire si une ampoule connectée se comporte normalement ou si elle est en train de transmettre tes habitudes de vie à un serveur inconnu.

3. La segmentation (Ne pas mettre tous ses œufs dans le même panier)
Dans un réseau classique, si un pirate entre sur votre imprimante, il peut souvent atteindre votre ordinateur personnel. La segmentation consiste à créer des murs virtuels pour que, même si un appareil est compromis, l'attaquant reste bloqué dans une zone isolée.

Glossaire technique
Sécurité & Concepts
Audit : Une inspection approfondie d'un système pour vérifier s'il est sécurisé et s'il respecte les règles de l'art.

Zero Trust (Confiance Zéro) : Une stratégie de sécurité qui part du principe qu'aucun appareil (même interne) n'est fiable par défaut. On vérifie tout, tout le temps.

Surface d'attaque : C'est l'ensemble des points (logiciels, ports, appareils) par lesquels un pirate pourrait tenter d'entrer dans ton réseau.

Vecteur d'attaque : Le chemin ou la méthode spécifique utilisé par un pirate pour accéder à une cible (ex: un mail de phishing, un port ouvert).

CVE (Common Vulnerabilities and Exposures) : Une liste publique de failles de sécurité connues. Chaque faille a son numéro (ex: CVE-2024-XXXX).

Hardening (Durcissement) : L'action de configurer un système pour le rendre plus résistant aux attaques (supprimer l'inutile, fermer les ports, etc.).

Remédiation : L'action de corriger une faille ou un problème de sécurité une fois qu'il a été détecté.

 Réseau & Analyse
Actifs (Assets) : Tous les équipements connectés au réseau (PC, smartphone, caméra, serveur).

Flux : Le mouvement des données entre deux points du réseau.

Périmètre : La limite de ton réseau (généralement ce qui est derrière ta box).

Fingerprinting (Empreinte numérique) : Technique pour deviner quel système d'exploitation ou quel logiciel tourne sur un appareil en analysant la façon dont il répond sur le réseau.

Capture de paquets (Sniffing) : L'action d'intercepter et d'enregistrer les petits morceaux de données (paquets) qui circulent sur le réseau pour les analyser.

DNS (Domain Name System) : L'annuaire d'Internet. Il transforme un nom (google.fr) en adresse IP (142.250.x.x). Analyser le DNS permet de voir quels sites tes appareils consultent.

VLAN (Virtual LAN) : Une méthode pour découper virtuellement ton réseau en plusieurs morceaux isolés (ex: un réseau pour les invités, un pour tes serveurs, un pour tes objets connectés).

 Menaces
Shadow IT : Logiciels ou appareils installés sur un réseau sans l'accord ou la connaissance de l'administrateur.

Mouvement Latéral : Quand un pirate réussit à entrer sur un appareil peu sécurisé (ex: une ampoule) et s'en sert pour "sauter" sur un appareil plus important (ton PC).

Exfiltration : L'action de voler des données et de les envoyer vers l'extérieur du réseau.

C2 (Command & Control) : Un serveur externe utilisé par un pirate pour diriger à distance des appareils infectés sur ton réseau.



