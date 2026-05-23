# 🔌 ZimaOS : Connecter et Automatiser sa clé USB Zigbee sur une VM Home Assistant

![ZimaOS](https://img.shields.io/badge/ZimaOS-%23000000.svg?style=for-the-badge&logo=linux&logoColor=white)
![KVM / Virsh](https://img.shields.io/badge/KVM__Virsh-%23FF6600.svg?style=for-the-badge&logo=virtualbox&logoColor=white)
![SSH](https://img.shields.io/badge/SSH-MobaXterm-%2341BDF5.svg?style=for-the-badge)

Ce guide rassemble toutes les commandes SSH et les scripts présentés dans la vidéo pour attacher de manière **permanente** et **automatique** une clé USB Zigbee (ex: Sonoff / Puce Silicon Labs) à une machine virtuelle sur ZimaOS.

> [!TIP]
> **Logiciel recommandé :** Pour vous connecter en SSH à votre serveur ZimaOS, je vous conseille d'utiliser **MobaXterm** (Windows) ou le Terminal natif (macOS/Linux).

---

## 🔍 1. Identification du matériel

Avant de commencer, vous devez identifier les identifiants uniques de votre configuration. Connectez-vous en SSH et lancez les commandes suivantes :

### Trouver le nom/ID de la machine virtuelle :
```bash
sudo virsh list --all
