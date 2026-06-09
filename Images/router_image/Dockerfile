# Image officielle FRR 
FROM frrouting/frr:latest

# Mise à jour et installation de busybox er iproute2 (pour les commandes ip complètes)
RUN apk update && apk add --no-cache busybox iproute2

# Activation des services exigés 
RUN sed -i 's/bgpd=no/bgpd=yes/' /etc/frr/daemons && \
    sed -i 's/ospfd=no/ospfd=yes/' /etc/frr/daemons && \
    sed -i 's/isisd=no/isisd=yes/' /etc/frr/daemons