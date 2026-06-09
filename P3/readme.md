# Projet BADASS - Partie 3 : Découverte de BGP avec EVPN

Ce dossier contient la configuration de la troisième et dernière étape du projet BADASS. L'objectif est d'automatiser notre réseau VXLAN. Plutôt que de configurer statiquement les tunnels, nous déployons BGP EVPN (sans MPLS).

BGP agit comme Control Plane pour découvrir dynamiquement les routeurs Leaf (VTEPs) et échanger automatiquement les adresses MAC des hôtes. Le routage de l'Underlay est assuré par OSPF, et BGP tourne sur l'AS 1.

# Topologie (Architecture Spine-Leaf)

- 1 Spine (Route Reflector) : snaggara_router-1. Il centralise et redistribue les routes BGP (ne fait pas de VXLAN).
- 3 Leafs (VTEPs) : snaggara_router-2, snaggara_router-3, snaggara_router-4.
- 3 Hôtes finaux : snaggara_host-1, snaggara_host-2, snaggara_host-3.

# Guide pas-à-pas

Suivez ces étapes dans l'ordre rigoureux du sujet pour valider le bon fonctionnement de l'infrastructure BGP EVPN.

1. Déploiement de l'infrastructure (Underlay & Overlay)

Pour démarrer proprement et éviter tout trafic parasite lors des premières vérifications, n'allumez que les routeurs sur GNS3. Laissez les nœuds des hôtes totalement éteints (stoppés) pour l'instant.

# A. Activation du Spine (Route Reflector)

Sur snaggara_router-1 :

vtysh -c 'configure terminal
! --- UNDERLAY (OSPF) ---
interface lo
 ip address 1.1.1.1/32
exit
interface eth0
 ip address 10.1.1.1/30
 ip ospf network point-to-point
exit
interface eth1
 ip address 10.1.1.5/30
 ip ospf network point-to-point
exit
interface eth2
 ip address 10.1.1.9/30
 ip ospf network point-to-point
exit
router ospf
 network 1.1.1.1/32 area 0
 network 10.1.1.0/30 area 0
 network 10.1.1.4/30 area 0
 network 10.1.1.8/30 area 0
exit

! --- OVERLAY (BGP EVPN) ---
router bgp 1
 bgp router-id 1.1.1.1
 neighbor 1.1.1.2 remote-as 1
 neighbor 1.1.1.2 update-source lo
 neighbor 1.1.1.3 remote-as 1
 neighbor 1.1.1.3 update-source lo
 neighbor 1.1.1.4 remote-as 1
 neighbor 1.1.1.4 update-source lo
 address-family l2vpn evpn
  neighbor 1.1.1.2 activate
  neighbor 1.1.1.3 activate
  neighbor 1.1.1.4 activate
  neighbor 1.1.1.2 route-reflector-client
  neighbor 1.1.1.3 route-reflector-client
  neighbor 1.1.1.4 route-reflector-client
 exit-address-family
exit
end
write memory'


# B. Activation des Leafs (VTEPs)

À titre d'exemple, voici ce qui est exécuté sur le Leaf 1 (snaggara_router-2) :
```
# Data Plane (Linux)
ip link add br0 type bridge
ip link set br0 up
ip link add vxlan10 type vxlan id 10 local 1.1.1.2 dstport 4789
ip link set vxlan10 master br0
ip link set vxlan10 up
ip link set eth1 up
ip link set eth1 master br0

# Control Plane (FRRouting)
vtysh -c 'configure terminal
interface lo
 ip address 1.1.1.2/32
exit
interface eth0
 ip address 10.1.1.2/30
 ip ospf network point-to-point
exit
router ospf
 network 1.1.1.2/32 area 0
 network 10.1.1.0/30 area 0
exit
router bgp 1
 bgp router-id 1.1.1.2
 neighbor 1.1.1.1 remote-as 1
 neighbor 1.1.1.1 update-source lo
 address-family l2vpn evpn
  neighbor 1.1.1.1 activate
  advertise-all-vni
 exit-address-family
exit
end
write memory'
```

(Procédez de même pour activer les Leafs 2 et 3 en adaptant leurs adresses IP respectives).

# 2. Vérification de l'Underlay (OSPF) et Peering BGP

Nous devons vérifier que le routage de base fonctionne avant de regarder le VXLAN.
Sur le VTEP 3 (snaggara_router-4), vérifiez la table de routage OSPF :

```
vtysh -c "show ip route"
```

Analyse : Vous devez voir les routes marquées d'un O (pour OSPF). Cela confirme que l'Underlay est fonctionnel et que les routeurs ont échangé leurs Loopbacks (ex: 1.1.1.1/32, 1.1.1.2/32).

Toujours sur le VTEP 3, vérifiez que la session BGP avec le Spine est établie :

```
vtysh -c "show bgp summary"
```

Analyse : Le résultat affiche deux tableaux, car notre session BGP transporte deux types de données distinctes (MP-BGP) :
- IPv4 Unicast : L'état (`State/PfxRcd`) affiche `0`. C'est normal, car nous n'utilisons pas BGP pour échanger des routes Internet classiques.
- L2VPN EVPN : C'est le cœur de notre projet. L'état doit afficher un chiffre (ex: `2`, `4`...) qui correspond au nombre de préfixes EVPN (VTEPs et MACs) reçus depuis le Spine.

# 3. La table EVPN sans les Hôtes (Routes de Type 3)

Puisque nous n'avons pas encore allumé les hôtes dans GNS3, les ports des routeurs ne reçoivent aucun signal. Le réseau virtuel est donc réellement "à vide" de toute machine, mais les tunnels existent déjà.

Sur un Leaf (ex: snaggara_router-4), vérifiez la table BGP :

```
vtysh -c "show bgp l2vpn evpn"
```

Analyse attendue : Seules des routes EVPN type-3 prefix sont visibles. Ces routes servent à l'auto-découverte des VTEPs. Il n'y a aucune route de Type 2 (liée à une adresse MAC), ce qui est parfaitement normal puisqu'aucun hôte n'a encore été démarré.

### 4. Magie EVPN : Apparition des MACs sans IP (Routes de Type 2)

C'est l'étape cruciale du sujet (Page 14). BGP EVPN peut découvrir un hôte sans même qu'on lui attribue d'IPv4.

Démarrez le nœud de l'Hôte 1 (snaggara_host-1) depuis l'interface de GNS3.
(Le simple fait de démarrer la machine virtuelle va allumer son interface réseau, ce qui pousse son système d'exploitation à envoyer automatiquement des trames IPv6 de découverte de voisinage sur le câble).

Observez la réaction sur le Leaf distant (ex: snaggara_router-2 ou 4) :

```
vtysh -c "show bgp l2vpn evpn"
```

Analyse attendue : Une nouvelle route EVPN type-2 prefix est apparue automatiquement ! Le switch virtuel du routeur a capté le trafic système natif (IPv6) de l'hôte lors de son démarrage, a extrait son adresse MAC et l'a distribuée dynamiquement via BGP à tout le réseau.

(Vous pouvez recommencer l'opération en démarrant l'hôte 2 pour voir une seconde route de Type 2 s'ajouter à la table).

# 5. Configuration IPv4 et Test de connectivité final

Pour vérifier que les données (Data Plane) circulent correctement de bout en bout, nous allons enfin configurer les IPs IPv4 sur nos hôtes (Overlay).

Sur chaque hôte, appliquez son IP respective :

# Sur snaggara_host-1
ip addr add 20.1.1.1/24 dev eth0
ip link set dev eth0 up

# Sur snaggara_host-2
ip addr add 20.1.1.2/24 dev eth0
ip link set dev eth0 up

# Sur snaggara_host-3
ip addr add 20.1.1.3/24 dev eth0
ip link set dev eth0 up


Le Ping inter-hôtes
Depuis snaggara_host-1, lancez un ping vers les autres machines :

ping 20.1.1.2 -c 4
ping 20.1.1.3 -c 4


Analyse finale : 1. Les pings passent avec 0% de perte.

# Wireshark
Si vous lancez une capture Wireshark sur les liens d'interconnexion (10.1.1.x), vous verrez que les paquets ICMP originaux (ex: 20.1.1.1 vers 20.1.1.2) sont parfaitement encapsulés dans des trames UDP VXLAN (port 4789). Le réseau SDN est 100% fonctionnel et scalable !