# Projet BADASS - Partie 2 : Découverte du VXLAN (Statique & Multicast)

Ce dossier contient la configuration de la deuxième étape du projet BADASS. L'objectif est de mettre en place un réseau VXLAN (Virtual eXtensible LAN) pour relier deux hôtes de manière transparente à travers un réseau intermédiaire, en utilisant d'abord une configuration statique, puis un groupe Multicast.

# Topologie
- 2 Hôtes finaux (host_snaggara-1 et host_snaggara-2).
- 2 Routeurs servant de VTEP (routeur_snaggara-1 et routeur_snaggara-2).
- 1 Switch central reliant les deux routeurs.
- Identifiant du VXLAN (VNI) : 10.
- Interface Bridge : br0.

# Étape 1 : Déploiement en Mode Statique (Unicast)
1. Démarrage
Allumez l'ensemble des machines de la topologie sur GNS3.

2. Configuration des Routeurs (VTEP)
Copiez-collez les scripts de configuration ci-dessous directement dans la console de chaque routeur.

Sur routeur_snaggara-1 :

```
# --- 1 : Configuration du réseau de transport (Underlay) ---
# On utilise un masque /30 car il n'y a que deux routeurs sur ce lien
ip addr add 10.1.1.1/30 dev eth1
ip link set dev eth1 up

# --- 2 : Création du tunnel VXLAN statique ---
# local 10.1.1.1 : On attache le tunnel à l'IP source de notre routeur
# remote 10.1.1.2 : On pointe explicitement vers l'IP du Routeur 2
ip link add name vxlan10 type vxlan id 10 local 10.1.1.1 remote 10.1.1.2 dstport 4789
ip link set dev vxlan10 up

# --- 3 : Configuration du domaine de diffusion local (Pont) ---
ip link set dev eth0 up
ip link add br0 type bridge
ip link set dev br0 up

# --- 4 : Liaison des interfaces (Le Pont virtuel) ---
ip link set dev eth0 master br0
ip link set dev vxlan10 master br0
(Sur routeur_snaggara-2, faites la même chose en inversant les adresses IP : 10.1.1.2/30 sur eth1, local 10.1.1.2 et remote 10.1.1.1).

```

3. Configuration des Hôtes
Assignez simplement les adresses IP du réseau virtuel (Overlay) aux deux hôtes finaux.


Sur host_snaggara-1 :

```
ip addr add 30.1.1.1/24 dev eth0
ip link set dev eth0 up
```
Sur host_snaggara-2 :

```
ip addr add 30.1.1.2/24 dev eth0
ip link set dev eth0 up
```

4. Le Test de Connectivité (Ping)
Une fois les configurations appliquées, il faut générer du trafic.
Sur host_snaggara-1 :

```
ping 30.1.1.2 -c 4
```
Résultat attendu : 0% packet loss. Le tunnel statique est fonctionnel !

5. Vérifications du mode Statique

Vérification A : Les détails du tunnel VXLAN statique
```
ip -d link show vxlan10
```
- vxlan id 10 : L'identifiant de notre réseau virtuel (VNI) est bien configuré.  - remote 10.1.1.2 : C'est la preuve du mode statique. 

Le tunnel pointe de manière fixe vers l'IP physique du routeur distant (Unicast). Il n'y a pas de groupe multicast ici.

Vérification B : La table MAC du Bridge (FDB)
```
bridge fdb show
```

En regardant les entrées de l'interface vxlan10, on observe que l'adresse MAC de l'hôte distant a été apprise et est directement associée à la destination explicite dst 10.1.1.2.

# Étape 2 : Passage en Mode Dynamique (Multicast)
Nous allons maintenant remplacer notre tunnel statique par un tunnel dynamique multicast.

1. Reconfiguration des Routeurs
Sur les deux routeurs, exécutez ces commandes pour modifier le VXLAN :

```
# On supprime l'ancien tunnel statique
ip link delete vxlan10

# On recrée le tunnel VXLAN en mode dynamique (Multicast)
# "remote" est remplacé par "group 239.1.1.1"
ip link add name vxlan10 type vxlan id 10 local 10.1.1.x group 239.1.1.1 dstport 4789
ip link set dev vxlan10 up

# On rattache le nouveau tunnel au bridge
ip link set dev vxlan10 master br0
(Remplacez 10.1.1.x par 10.1.1.1 sur le routeur 1 et 10.1.1.2 sur le routeur 2).
```

2. Le Test de Connectivité Multicast
Refaites un ping depuis host_snaggara-1 vers host_snaggara-2. Les routeurs vont utiliser l'adresse multicast pour se découvrir dynamiquement.

# Étape 3 : Vérifications & Explications des Résultats
Pour valider le bon fonctionnement de l'architecture, effectuez les vérifications suivantes sur les routeurs.

Vérification A : Les détails du tunnel VXLAN
Commande : ip -d link show vxlan10

- vxlan id 10 : L'identifiant de notre réseau virtuel (VNI) est bien pris en compte.
- group 239.1.1.1 : Confirme que le tunnel utilise le groupe multicast.
- dstport 4789 : Utilisation du port de destination UDP standard assigné par l'IANA.

Vérification B : La table MAC du Bridge (FDB)
Commande : bridge fdb show

Analyse : Le bridge agit comme un véritable switch de niveau 2 apprenant les emplacements des machines :

Il a appris que la machine locale (host_snaggara-1) se trouve derrière le câble physique eth0.

Il a appris que la machine distante (host_snaggara-2) se trouve derrière l'interface virtuelle vxlan10. Le routeur sait désormais que pour joindre cette adresse MAC, il doit encapsuler la trame dans le tunnel VXLAN.

# Étape 4 : Analyse de trames avec Wireshark
Pour prouver concrètement l'encapsulation, lancez une capture Wireshark sur le câble reliant routeur_snaggara-1 au Switch_snaggara et relancez un ping depuis host_snaggara-1.

Analyse du paquet intercepté :

- Couche Externe (Underlay / Ethernet II & IPv4) : On voit les adresses MAC et IPs physiques des routeurs (ex: Src: 10.1.1.1, Dst: 10.1.1.2 lors d'un échange connu, ou Dst: 239.1.1.1 pour la découverte multicast).

- Couche Transport (UDP) : Le paquet utilise un Src Port aléatoire et le Dst Port: 4789 (Standard VXLAN).

En-tête VXLAN : On observe distinctement le VXLAN Network Identifier (VNI): 10.

Couche Interne (Overlay / ICMP) : À l'intérieur du VXLAN, on retrouve la trame Ethernet d'origine avec les IPs de nos hôtes finaux (Src: 30.1.1.1, Dst: 30.1.1.2) et le paquet ICMP Echo Request/Reply.