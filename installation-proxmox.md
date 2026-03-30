# Installation de proxmox

Parfait ! La base Debian 13.2 est prête, le réseau est bridgé en vmbr0 et le FQDN est bien résolu. On passe désormais au cœur de la mission : l'installation de Proxmox 9.1 (portage ARM pour ton Raspberry Pi 5).

Puisque nous sommes sur une architecture ARM64, nous utilisons les dépôts spécifiques à ce portage.

## 1. Ajout du dépôt Proxmox 9 (Pimox)

Tout d'abord nous allons importer la clé GPG:

```Bash
curl -s https://raw.githubusercontent.com/pimox/pimox9/master/KEY.gpg | gpg --dearmor -o /etc/apt/trusted.gpg.d/pimox-archive-keyring.gpg
```
