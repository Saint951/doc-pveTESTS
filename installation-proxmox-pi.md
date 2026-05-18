# Installation de proxmox

Parfait ! La base Debian 13.2 est prête, le réseau est bridgé en vmbr0 et le FQDN est bien résolu. On passe désormais au cœur de la mission : l'installation de Proxmox 9.1 (portage ARM pour ton Raspberry Pi 5).

Puisque nous sommes sur une architecture ARM64, nous utilisons les dépôts spécifiques à ce portage.

## 1. Installation de proxmox sur le pi (PXVIRT)

Tout d'abord nous importons la clé gpg du dépot.

```bash
curl -L https://mirrors.lierfang.com/pxcloud/lierfang.gpg -o /etc/apt/trusted.gpg.d/lierfang.gpg
```

Ajout des sources Proxmox (PXVIRT)

L'installation de Proxmox 9.1 sur Raspberry Pi repose sur le dépôt PXVIRT, hébergé sur les miroirs de Lierfang. Ce dépôt fournit les binaires spécifiquement compilés pour l'architecture aarch64. L'utilisation de variables d'environnement système permet d'automatiser cette étape et d'assurer la pérennité de la configuration du nœud pveTESTSpi.

```bash
source /etc/os-release
```

Ajout du mirroir de Lierfang dans les sources de Pi

```Bash
echo "deb [https://mirrors.lierfang.com/pxcloud/pxvirt](https://mirrors.lierfang.com/pxcloud/pxvirt) $VERSION_CODENAME main" | sudo tee /etc/apt/sources.list.d/pxvirt-sources.list
```

Nous pouvons donc désormais installer proxmox grâce à apt:

```Bash
apt install proxmox-ve pve-manager qemu-server pve-cluster
```
