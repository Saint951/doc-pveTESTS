# Documentation de l'installation de proxmox sur raspberry pi 5.

Ce document détaille la mise en place d'un nœud de virtualisation expérimental basé sur l'architecture ARM.
L'objectif est d'évaluer les capacités du Raspberry Pi 5 en tant qu'hyperviseur via le portage de Proxmox VE.
Ce serveur, identifié sous le nom de pveTESTSpi, servira d'environnement de test pour le déploiement de micro-services et de conteneurs LXC au sein de l'infrastructure cub.corsica.

# Cahier des charges

 ```
 pveTESTSpi Modèle : raspberry pi 5

FQDN : pveTESTSpi.cub.corsica

IP : 192.168.92.23/24 sur vmbr0

Passerelle : 192.168.92.254 
```
