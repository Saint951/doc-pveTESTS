# Documentation de l'installation de proxmox sur raspberry pi 5.

La gestion d'un cluster Proxmox nécessite un minimum de trois nœuds pour assurer la haute disponibilité et la cohérence des votes (quorum).
Pour optimiser les coûts et l'espace au sein du local technique de CUB, le choix a été fait d'intégrer des Raspberry Pi 5 comme nœuds légers au sein des clusters.
Ce rapport présente la configuration du nœud pveTESTSpi et son intégration dans le cluster « pveTESTS ».
L'enjeu est de valider la viabilité de cette architecture hybride ARM/x86 pour tester les nouveaux workflows de déploiement automatisé via Cloud-Init et la gestion des droits utilisateurs.

>[!TIP]
>Certaines commandes dans cette documentation nécessitent d'être l'utilisateur root ou de sudo en tant que sudoer.

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

Cette commande devrait renvoyer le nom d'hôte de vous avez mis, cependant Proxmox est extrêmement sensible à la résolution de nom. Le nom d'hôte doit être lié à votre adresse IP statique dans le fichier `/etc/hosts`.

À l'aide de votre éditeur de text préferer (`nano` ou `vi` etc..)

```Bash
vi /etc/hosts`
```

Le fichier devrait contenir ceci:

```conf
127.0.0.1       localhost
255.255.255.255 broadcastost
::1             localhost

ff02::1   ip6-allnodes
ff02::2   ip6-allrouters
```

Il faut rajouter la ligne suivantes:

```conf
127.0.0.1       localhost
[IP_du_PI]      [nom_hôte et/ou nom_complet]
```

Maintenant vous pouvez confirmer et fermer.

>[!TIP]
>`esc` puis `:wq` avec vi,vim,nvim.
>
>`ctrl + X `ensuite `Y` puis `backspace` avec nano.


## 2. Configuration réseau (Le Bridge vmbr0)

Comme spécifié dans le cahier des charges, Proxmox utilise un pont (bridge) pour permettre aux VM et conteneurs LXC de sortir sur le réseau physique de CUB.

Sur Debian 13, on utilise généralement ifupdown2. Éditez votre fichier d'interfaces :

```bash
vi /etc/network/interfaces
```

Il faut y mettre:

```conf
auto lo
iface lo inet loopback

# Interface physique du Raspberry Pi 5
iface eth0 inet manual

# Pont Proxmox pour le cluster [nom_cluster]
auto vmbr0
iface vmbr0 inet static
        address [ip_pi]/[masque]
        gateway [ip_passerelle]
        bridge-ports [interface_physique]
        bridge-stp off
        bridge-fd 0
```

>[!IMPORTANT]
>Si vous utilisez NetworkManager (par défaut sur certaines images Pi), il est fortement recommandé de le désactiver pour laisser Proxmox gérer le réseau via les fichiers de configuration standards :
>```Bash
>sudo systemctl disable --now NetworkManager
>```

## 3. Mise à jour et Préparation des dépôts

Tout d'abord nous allons rendre possible l'accès à internet:

```Bash
# 1. On active l'interface physique
sudo ip link set eth0 up

# 2. On lui donne ton IP statique
sudo ip addr add 192.168.92.23/24 dev eth0

# 3. On définit la passerelle pour sortir sur le web
sudo ip route add default via 192.168.92.254
```

Maintenant nous testons la connexion:

```Bash
ping -c 3 google.com
```

Avant de lancer l'installation de Proxmox 9.1, assurez-vous que votre système est parfaitement à jour sur cette branche 13.2 :

```Bash
sudo apt update && sudo apt full-upgrade -y
```

installation de package nécessaires:

```Bash
apt install bridge-utils ifupdown2 -y
```
>[!IMPORTANT]
>Pour que le pi est accès au cluster, on a mis en place un script au démarrage qui reconfigure son ip.
>```bash
>#on crée le script dans ce fichier
>sudo nano /usr/local/bin/config-reseau.sh
>```
>```bash
>#!/bin/bash
># on active l'interface physique
>ip link set ethO up
>
># on lui attribut l'ip statique (on enlève aussi les anciennes conf)
>ip addr flush dev eth0
>ip addr add 192.168.92.23/24 dev eth0
>
># on met la passerelle par défaut
>ip route add default via 192.168.92.254
>```
>on rend alors le script exécutable:
>```bash
>sudo chmod +x /usr/local/bin/config-reseau.sh
>```
>
>on crée un fichier service:
>```bash
>sudo nano /etc/systemd/system/config-reseau.service
>```
>et on ajoute cette configuration:
>```bash
>[Unit]
>DescriptionConfiguration Réseau Statique Persistante
>After=network.target
>
>[Service]
>Type=oneshot
>ExecStart=/usr/local/bin/config-reseau.sh
>emainAfterExit=yes
>
>[Install]
>WantedBy=multi-user.target
>```
>Enfin on active le service au démarrage:
>```bash
>sudo systemctl daemon-reload
>
>sudo systemctl enable config-reseau.service
> ```


>[!TIP]
>Cela était necessaire puisqu'il faut désactiver network manager car sinon il prenait la main sur la carte réseau du pi.

Et voilà, votre pi devrait être prêt pour la suite. 🎊
