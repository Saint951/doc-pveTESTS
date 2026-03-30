# Documentation de l'installation de proxmox sur raspberry pi 5.

La gestion d'un cluster Proxmox nécessite un minimum de trois nœuds pour assurer la haute disponibilité et la cohérence des votes (quorum).
Pour optimiser les coûts et l'espace au sein du local technique de CUB, le choix a été fait d'intégrer des Raspberry Pi 5 comme nœuds légers au sein des clusters.
Ce rapport présente la configuration du nœud pveTESTSpi et son intégration dans le cluster « pveTESTS ».
L'enjeu est de valider la viabilité de cette architecture hybride ARM/x86 pour tester les nouveaux workflows de déploiement automatisé via Cloud-Init et la gestion des droits utilisateurs.

# Cahier des charges

 ```
 pveTESTSpi Modèle : raspberry pi 5

FQDN : pveTESTSpi.cub.corsica

IP : 192.168.92.23/24 sur vmbr0

Passerelle : 192.168.92.254 
```
