# BADASS

Un projet réseau du master architecte des systèmes d'information à 42 (réseaux et sécurité). L'objectif : construire, dans GNS3, un réseau de data center en architecture **Spine-Leaf**, avec des routeurs et des hôtes qui sont nos propres conteneurs Docker, puis y déployer VXLAN et BGP EVPN.

## Les parties

```text
Images/   les deux images Docker : un hôte minimal (Alpine) et un routeur (FRRouting)
P1/       validation des images dans GNS3 : un hôte, un routeur, un ping
P2/       VXLAN statique puis multicast entre deux leafs
P3/       BGP EVPN : découverte automatique des VTEPs, échange des MAC, OSPF en underlay
```

Chaque dossier contient le projet GNS3 (`.gns3project`), les configurations des routeurs et des hôtes, et un `readme.md` qui explique la topologie et les vérifications faites.

## Ce que le projet travaille

- Construction d'images Docker légères pour du réseau (FRR avec bgpd, ospfd, zebra)
- VXLAN : encapsulation L2 sur L3, VNI, VTEP
- BGP EVPN comme plan de contrôle, sans MPLS
- OSPF pour l'underlay
- Lecture et débogage de tables de routage, de MAC et de tunnels

## Reproduire

```bash
cd Images/router_image && docker build -t router_snaggara .
cd ../host_image && docker build -t host_snaggara .
```

Puis importer le projet `.gns3project` de la partie voulue dans GNS3, avec les deux images déclarées comme templates Docker.
