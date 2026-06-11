# Projet BADASS - Partie 1 : Déploiement Initial & Validation

Ce dossier contient la configuration de la première étape du projet BADASS. L'objectif de cette partie est de valider la création de nos propres images Docker et de s'assurer de leur bon fonctionnement au sein de l'environnement GNS3.

## Présentation des Images Docker et de leurs Services

Pour ce projet, nous avons construit deux images distinctes. Même si cette première partie ne teste qu'un simple `ping`, les images sont déjà préparées avec les services nécessaires pour supporter l'architecture Spine-Leaf complexe des parties suivantes.

### 1. L'Image Hôte (`host_snaggara`)
L'hôte représente un serveur final "bête" dans notre Data Center. Pour coller aux exigences de légèreté d'un environnement de production, il doit être le plus minimaliste possible.

* **OS de base :** Alpine Linux (ou Ubuntu minimal)
* **Services/Paquets installés :**
  1. **BusyBox :** C'est le couteau suisse de l'image. Il regroupe en un seul exécutable ultra-léger toutes les commandes standards UNIX (comme `sh`/`ash` pour le terminal, `ping`, `echo`, `cat`). Il permet d'avoir un système fonctionnel sans charger l'image avec des utilitaires lourds.
  2. **iproute2 :** Installé dès la Partie 1 pour configurer l'IP basique (`ip addr`), ce paquet est surtout indispensable pour la suite du projet. C'est lui qui permettra, dans les parties suivantes, de manipuler les interfaces réseaux avancées (liens logiques, routage fin, etc.) sans dépendre des outils obsolètes comme `ifconfig`.

* **Pourquoi ?** Cette combinaison permet d'allier la légèreté absolue de BusyBox pour les outils de tous les jours, tout en garantissant la présence d'`iproute2` pour piloter le réseau de manière moderne dès que le projet va se complexifier.

### 2. L'Image Routeur (`router_snaggara`)
Le routeur est l'équipement intelligent du réseau. Il servira plus tard de VTEP (VXLAN) et de routeur BGP/OSPF.
* **OS de base :** Alpine Linux (ou Ubuntu)
* **Services/Paquets installés :**
  * `frr` (FRRouting) : C'est la suite de routage (le daemon) qui permet de transformer notre conteneur Linux en un vrai routeur matériel. Il inclut les services OSPF et BGP.
  * `iproute2` : Pour la création des bridges et des tunnels VXLAN directement dans le noyau Linux.
  * `tcpdump` (Optionnel mais recommandé) : Pour analyser le trafic et débugger les paquets encapsulés plus tard.
* **Pourquoi ?** C'est le cœur du projet. FRRouting nous offre le *Control Plane* (via le shell `vtysh`), tandis que Linux gère le *Data Plane*.

---

## 🛠️ Guide de Déploiement

La topologie est extrêmement simple : un câble direct relie l'Hôte (`eth0`) au Routeur (`eth0`).

### Étape 1 : Démarrage
1. Importez les deux conteneurs dans GNS3.
2. Reliez-les via un câble réseau standard.
3. Démarrez les deux machines.

### Étape 2 : Configuration IP
Appliquez les configurations réseau de base via la console de chaque machine.

**Sur l'Hôte :**
\`\`\`bash
ip addr add 10.1.1.1/24 dev eth0
ip link set eth0 up
\`\`\`

**Sur le Routeur :**
\`\`\`bash
ip addr add 10.1.1.2/24 dev eth0
ip link set eth0 up
\`\`\`

---

## 🔍 Validation (Le Test du Ping)

Une fois les adresses IP configurées, la validation s'effectue par un simple test de connectivité.

Depuis le terminal de l'**Hôte**, lancez :
\`\`\`bash
ping 10.1.1.2 -c 4
\`\`\`

**✅ Résultat attendu :** Les 4 paquets doivent passer (`0% packet loss`). Cela prouve que :
1. Nos images Docker sont fonctionnelles.
2. L'intégration dans GNS3 est réussie.
3. Les interfaces réseaux virtuelles communiquent correctement au niveau 2 et niveau 3.

Nous sommes prêts à passer à la Partie 2 !