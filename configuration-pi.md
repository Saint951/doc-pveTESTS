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

# Préparation du raspberry pi 5

>[!IMPORTANT]
>Cette documentation n'est pas garantie de fonctionner sur debian <13.2;
>Pour consulter la version:
> ```Bash
> cat /etc/debian_version
> ```

## 1. Configuration d'un nom d'hôte

Comme nous souhaitons nommer notre pi: "pveTESTpi.corsica", Il va falloir adapter la commande suivante:

```Bash
sudo hostnamectl set-hostname [HOSTNAME]
```

Pour vérifier si la configuratrion a fonctionné vous pouvez executer la commande:

```Bash
hostname
```

Cette commande devrait renvoyer le nom d'hôte de vous avez mis, cependant Proxmox est extrêmement sensible à la résolution de nom. Le nom d'hôte doit être lié à ton adresse IP statique dans le fichier `/etc/hosts`.
