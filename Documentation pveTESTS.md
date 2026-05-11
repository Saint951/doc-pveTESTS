# Documentation pveTESTS


## Contexte du projet


Le projet PveTESTS a été initié pour répondre à un besoin critique de sécurisation de notre infrastructure.
L'objectif principal est de fournir un environnement de "bac à sable" (sandbox) totalement isolé du cluster de production, évitant ainsi tout risque d'interruption de service lors de phases expérimentales.

Ce nouveau cluster de test Proxmox se concentre sur deux axes majeurs:


1. La sécurité opérationnelle : Disposer d'un droit à l'erreur total sans impacter les services critiques
2. L'automatisation : Valider l'implémentation et le comportement des templates via Cloud-init pour l'industrialiser nos déploiments.


Pour garantir une expérience de test fluide et représentative, nous avons exploité l'inventaire matériel réalisé lors de notre précédente mission.
Ce cluster a été bâti en sélectionnant rigoureusement les serveurs disposant des meilleures ressources en CPU et en RAM parmi le parc disponible.


### Machines sélectionnées

1. pveTESTS1 : Sérial Number : 7GFGP4J ; Serveur HP avec 32Go de RAM, processeurs: 2 Intel Xeon x5560 et 900 + 146 Go de stockage
2. pveTESTS2 : Sérial Number : 2QD6V3J ; Serveur HP avec 32Go de RAM, processeurs: 2 Intel Xeon E5450 et 900 + 146 Go de stockage

### Matériel fourni (addition)

> pveTESTSpi: Raspberry pi 5


## Configuration Réseau


Le cluster pveTESTS est isolé sur le VLAN 92 (management-proxmox).
Cette section détaille la configuration des switchs pour propager ce flux jusqu'aux serveurs en salle 203.

1. Interconnexion des Switchs (Trunking)

la liaison entre le cœur de réseau et le switch de distribution est déjà opérationnelle:

> SW-CORE: Le VLAN 92 est propagé via le lien 802.1q sur le port 44 (configuré en ALL Tagged).
> SW-CUB: Reçoit le flux sur le port 47.

2. Configuration de SW-CUB (Salle 203)

Sur le switch SW-CUB, le VLAN 92 a été crée et assigné aux ports de desserte pour les serveurs.

Actions effectuées:

> Création du VLAN : ID 92 | Nom management-proxmox
> Configuration des ports d'accès: Les ports ont été configurés en mode UNTAGGED (Access) pour permettre aux serveurs de communiquer sans configuration de VLAN interne au niveau de l'OS.


## Configuration des serveurs:


### Initialisation des disques virtuelles:

Configuration de "Systeme" de 146Go en RAID0 et de "zpool" de 900Go en RAID0 depuis le BIOS sur les deux serveurs.


## Installation de Proxmox:


### Sur pveTESTS1:

```text
login: root
mdp: Cluster@!Test2B
email: admin@cub.corsica
FQDN: pveTESTS1.cub.corsica

vmbr0 (eno1)
IP: 192.168.92.21/24
Passerelle: 192.168.92.254
Liaison: port 6 de SW-CORE (VLAN 92)

vmbr2 (eno2) Vlan aware 50, 60 et 70
Liaison: port 26 de SW-CORE (VLAN 50 et 60)
```


### Sur pveTESTS2

```text
login: root
mdp: Cluster@!Test2B
email: admin@cub.corsica
FQDN: pveTESTS2.cub.corsica

vmbr0 (nic0):
IP: 192.168.92.22/24
Passerelle: 192.168.92.254
Liaison: port 8 de SW-CORE (VLAN 92)

vmbr2 (nic1) Vlan aware 50, 60 et 70
Liaison: port 27 de SW-CORE (VLAN 50, 60 et 70)
```


### Sur pveTESTSpi

```text
login: root
mdp: Cluster@!Test2B
email: admin@cub.corsica
FQDN: pveTESTSpi.cub.corsica
IP: 192.168.92.23/24
Passerelle: 192.168.92.254
Liaison: port 13 de SW-CORE (VLAN92)
```


## Actualisation des certificats SSL


Afin de garantir un fonctionnement optimal et de pouvoir facilement configurer les différents éléments du cluster depuis n’importe quel élément, il est nécessaire d’actualiser les certificats SSL de tous les équipements avec les commandes suivantes:

```bash
pvecm updatecerts -F
systemctl restart pvedaemon pveproxy
```


> Conseil:
> En cas de perte du mot de passe

Méthode via GRUB


Redémarrer le serveur
Choisir Advanced options
(recovery mode) → appuyer sur e
Trouver la ligne commençant par linux
Ajouter à la fin :

```bash
init=/bin/bash
```

Démarrer

```bash
mount -o remount,rw /
passwd
exec /sbin/init
reboot -f
```

# Fonctionnement de Cloud-init

Cloud-init est l'outil standard pour automatiser l'initialisation des machines virtuelles (VM) Linux lors de leur premier démarrage.
Il permet de personnaliser une instance (utilisateurs, paquets, fichiers) sans intervention manuelle. 
Voici un résumé de son fonctionnement et des distinctions clés à maîtriser.  
Les 5 étapes du bootCloud-init n'exécute pas tout en même temps. 
Il suit une séquence précise pour s'assurer que les ressources (comme le réseau) sont prêtes avant d'appliquer les configurations.  
Detect (ds-identify) : Identifie l'environnement cloud (AWS, GCP, OpenStack, etc.) pour savoir où chercher les données.

> Local : S'exécute avant le réseau. C'est ici que la configuration réseau est appliquée.  
> Network : Une fois le réseau actif, il récupère le user-data et le metadata depuis le fournisseur.  
> Config : Le cœur de l'action. Il crée les utilisateurs, installe les paquets et écrit les fichiers.  
> Final : Exécute les scripts utilisateur (runcmd) et signale la fin de l'initialisation.  bootcmd vs runcmdLa confusion entre ces deux directives est fréquente, mais leur usage diffère radicalement selon le moment et la fréquence d'exécution.


## user-data vs network-config


C'est la règle d'or de cloud-init : le réseau ne se configure pas dans le user-data.

User-dataContenu : Utilisateurs, clés SSH, paquets à installer (packages), fichiers à créer (write_files) et scripts de fin (runcmd).  
Format : Doit impérativement commencer par #cloud-config sur la première ligne.  
Accessibilité : Souvent lisible depuis la VM via le service de métadonnées (ex: http://169.254.169.254), donc n'y stockez jamais de secrets sensibles.  
Network-configContenu : Configuration des interfaces, IP statiques, routes et DNS.  
Origine : Fournie par le fournisseur cloud ou via une source de données séparée (comme le mode NoCloud).  
Rôle : Elle est traitée durant le stage Local, bien avant que le user-data ne soit même téléchargé.  

Conseils de survie

Idempotence : Si vous utilisez cloud-init clean pour rejouer une config, assurez-vous que vos scripts peuvent être lancés plusieurs fois sans tout casser.  
Validation : Utilisez toujours cloud-init schema -c user-data.yaml --annotate avant de déployer pour éviter les erreurs de syntaxe YAML.  Débogage : En cas de problème, le fichier de log principal est /var/log/cloud-init.log.  


## Gestion des droits sur Proxmox


### Créer l'utilisateur dans Proxmox

La première étape va être de créer un compte.  
Depuis la navigation dans la console, aller sur Datacenter > Permissions > Users. 
On arrive sur la liste des utilisateurs.  

> Cliquer sur le bouton Add.
> Commencer par rentrer l'identifiant de l'utilisateur, choisir le Realm Proxmox VE authentification Server.  
> Puis saisir le mot de passe de l'utilisateur et cliquer sur Add.  
> Note : On va choisir l'authentification Proxmox VE authentification Server et non PAM. 
> Pour utiliser le Realm PAM, l'utilisateur doit être créé au niveau du système, ce qui n'est pas nécessaire car on souhaite seulement un accès à l'interface Web de Proxmox.


### Les privilèges


Les Rôles regroupent des autorisations sous forme de privilèges qu'on peut appliquer soit à des groupes soit à des utilisateurs. Proxmox a des rôles prédéfinis disponibles de base dans Datacenter > Permissions > Roles.
On peut néanmoins créer des rôles en appuyant sur "Create" en haut à gauche pour pouvoir peaufiner les autorisations selon les besoins.

> Liste des privilèges :
>

> **VM / Conteneurs (Guest)**
> 
> VM.Allocate : Créer ou supprimer des VM sur un storage.  
> VM.Config.Options : Modifier tout sauf le CPU, la RAM et le réseau (ex: boot order).  
> VM.Config.CPU / RAM / Network / Disk : Modifier spécifiquement ces ressources.  
> VM.Config.HWType : Changer le type de matériel émulé.  
> VM.Config.CDROM : Éjecter ou insérer un ISO.  
> VM.Config.Cloudinit : Configurer les paramètres Cloud-init.  
> VM.Console : Accéder à la console interactive de la VM.  
> VM.PowerMgmt : Démarrer, arrêter, reset ou suspendre.  
> VM.Migrate : Déplacer la VM vers un autre nœud du cluster.  
> VM.Backup : Créer des sauvegardes (Backups).  
> VM.Snapshot : Créer et supprimer des snapshots.  
> VM.Snapshot.Rollback : Revenir à un état précédent via snapshot.  
> VM.Clone : Cloner une VM ou un template.  
> VM.Audit : Voir la configuration de la VM sans pouvoir la modifier.  
>
> 
> Stockage (Datastore)
> Datastore.Allocate : Créer/supprimer des volumes de stockage, allouer de l'espace.  
> Datastore.AllocateSpace : Créer des disques virtuels sur le stockage.  
> Datastore.AllocateTemplate : Autoriser l'upload de templates ou d'ISOs.  
> Datastore.Audit : Voir l'état du stockage et le contenu.  
>
> 
> Réseau & SDN
> 
> SDN.Allocate : Créer ou configurer des réseaux définis par logiciel.  
> SDN.Audit : Voir la configuration SDN.  
> SDN.Use : Utiliser un bridge ou un VNet sur une interface réseau de VM.  
> 
> 
> Système & Nœud (Sys)
> 
> Sys.Console : Accès au shell (ligne de commande) du serveur hôte.  
> Sys.Audit : Voir les logs système et l'état du matériel (CPU, température, etc.).  
> Sys.Syslog : Consulter les journaux système (syslog).  
> Sys.PowerMgmt : Éteindre ou redémarrer le serveur physique (le nœud).  
> Sys.Modify : Modifier les paramètres réseau du nœud ou le temps système.  
> 
> 
> Accès & Utilisateurs (Permissions)
> 
> Permissions.Modify : Modifier les droits d'accès des autres (attention, droit très critique).  
> User.Modify : Créer, supprimer ou modifier des utilisateurs.  
> Group.Allocate : Créer ou supprimer des groupes d'utilisateurs.  
> Realm.Allocate : Gérer les méthodes d'authentification (AD, LDAP, PAM).  
> Realm.AllocateUser : Assigner des utilisateurs à un domaine spécifique.  
> DiversPool.Allocate : Créer ou gérer des pools de ressources.  
> Mapping.Modify / Use / Audit : Gérer les correspondances de matériel (USB/PCI passthrough).  
> VM.GuestAgent.FileRead / Write / Exec : Interagir avec les fichiers à l'intérieur de la VM via l'agent QEMU.  
 
### Permissions sur une VM

On peut autoriser l'accès à certaines VM selon les rôles via: 
[VM concerné] > Permissions > Add  

## Gérer les images OCI sur Proxmox

### Importer des images OCI

1. Allez dans votre stockage (par exemple : local).  
2. Cliquez sur la section CT Templates, comme pour les templates LXC classiques.  
3. Cliquez sur le bouton Pull from OCI Registry situé à côté des autres boutons (si vous ne le voyez pas, c'est que vous devez mettre à jour PVE).  
   
Vous devez ensuite compléter le formulaire pour spécifier l'image OCI à télécharger.
Dans le champ "Reference", saisissez le nom de l'image de façon explicite. 
Utilisez la syntaxe proprietaire/image ou repository/proprietaire/image.

> Exemple : louislam/uptime-kuma.  
> Pour Docker Hub : docker.io/louislam/uptime-kuma.  
> Pour GitHub (GHCR) : ghcr.io/louislam/uptime-kuma.  

Vous devez ensuite cliquer sur Query tags, et si l'image a été trouvée, la liste des tags (et versions) sera affichée.
Pour cet exemple, sélectionnez le tag 2 pour récupérer la dernière version de la V2 d'Uptime Kuma.  
Lancez le téléchargement et patientez un instant.
L'image OCI apparaît dans la bibliothèque aux côtés des autres modèles déjà téléchargés.

## Création et configuration du conteneur

La création du conteneur est très similaire à celle d'un LXC classique.
Une fois l'image téléchargée, cliquez sur le bouton Créer CT.

### Onglet Général

Donnez un ID, par exemple 303, et un nom comme lxc-oci-uptime-kuma.  
Décochez "Unprivileged container" si vous rencontrez des soucis de droits, bien que dans notre cas, Uptime Kuma fonctionnera bien dans un conteneur non-privilégié.

### Onglet Template

Sélectionnez l'image louislam/uptime-kuma que vous venez de télécharger, même si elle apparaît avec le nom de l'archive.

### Onglets Disques

> CPU > Mémoire

Allouez les ressources.
Ici, il convient d'adapter en fonction des besoins de l'application.  
Uptime Kuma avec peu de sondes n'a pas besoin de beaucoup de ressources.  
Le fait d'associer un disque au conteneur implique que son stockage sera persistant.

### Onglet Réseau

La mise en réseau du conteneur est différente selon si l'image OCI est exécutée via LXC ou Docker. 
En effet, avec Docker, vous feriez un mappage de port (-p 3001:3001), tandis qu'avec LXC, le conteneur a sa propre adresse IP sur votre réseau (via le pont vmbr0 ou une autre interface sélectionnée).  
Vous avez le choix entre le DHCP ou la configuration d'une adresse IP statique.  
Configurez le DNS si nécessaire.  
Confirmez pour lancer la création du conteneur et patientez le temps que l'opération se termine côté Proxmox.  
Personnaliser les variables d'environnementAvant d'exécuter un conteneur, vous pouvez personnaliser sa configuration en ajustant les variables d'environnement. 
En effet, dans le cas de l'utilisation d'une image OCI sur Proxmox VE en mode natif, il n'est pas possible de jouer sur les arguments de la commande docker run ou d'utiliser les directives d'un Docker Compose. 
De ce fait, c'est via cette méthode que l'on peut tenter de personnaliser la configuration d'une application.  
Dans la section Options du conteneur, double-cliquez sur Environment.  
Une fenêtre avec les variables d'environnement définies dans l'image OCI va s'afficher. 
Vous pouvez en configurer d'autres, en fonction de ce qui est pris en charge par l'application déployée (consultez la documentation de la solution en question).  
Dans le cas d'Uptime Kuma, il existe de nombreuses variables d'environnement. 
Par exemple, la variable UPTIME_KUMA_PORT sert à personnaliser le port pour accéder à l'interface Web. 
Cela pourrait permettre de préciser 3002 à la place de 3001 (port par défaut). 
Une fois vos modifications effectuées, validez.
C'est éditable à tout moment, mais la prise en charge des nouvelles valeurs implique un redémarrage du conteneur.
Premier démarrage du conteneurEffectuez un clic droit sur le conteneur pour le démarrer via le bouton Start.  
Cliquez sur l'onglet Network pour afficher l'adresse IP du conteneur. 
Ce sera utile pour récupérer l'adresse IP dans le cas où la carte réseau est configurée en DHCP. 
De plus, sur ce type de conteneurs applicatifs, vous n'avez pas toujours un shell interactif complet, donc l'onglet Console ne sera pas d'une grande utilité (cela varie d'une image à l'autre).  
Ouvrez un navigateur et spécifiez l'adresse IP du conteneur suivie du port 3001 (ou 3002 si la variable avec le port personnalisé a été appliquée). 