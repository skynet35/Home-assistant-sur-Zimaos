# 🔌 ZimaOS : Connecter et Automatiser sa clé USB Zigbee sur une VM Home Assistant

![ZimaOS](https://img.shields.io/badge/ZimaOS-%23000000.svg?style=for-the-badge&logo=linux&logoColor=white)
![KVM / Virsh](https://img.shields.io/badge/KVM__Virsh-%23FF6600.svg?style=for-the-badge&logo=virtualbox&logoColor=white)
![SSH](https://img.shields.io/badge/SSH-MobaXterm-%2341BDF5.svg?style=for-the-badge)

Ce guide rassemble toutes les commandes SSH, configurations XML et scripts présentés dans la vidéo pour attacher de manière **permanente** et **automatique** une clé USB Zigbee (ex: Sonoff) à une machine virtuelle HAOS sur ZimaOS.

> [!TIP]
> **Logiciel recommandé :** Pour vous connecter en SSH à votre serveur ZimaOS, je vous conseille d'utiliser **MobaXterm** (Windows) ou le Terminal natif (macOS/Linux).

---

## 🎯 Le Concept Fonctionnel
ZimaOS réinitialise les connexions USB à chaque redémarrage de l'hôte physique. Ce script permet d'écouter l'état de la machine virtuelle et d'y ré-injecter à chaud la clé USB dès que le système est prêt.

---

### 📦 Commandes SSH et Scripts de Configuration
<details>
<summary><b>▶ Cliquez ici pour dérouler l'intégralité des commandes de la vidéo</b></summary>

```bash
# ==============================================================================
# ÉTAPE 1 : IDENTIFICATION DU MATÉRIEL
# ==============================================================================

# Trouver le nom / l'ID unique de votre Machine Virtuelle :
sudo virsh list --all
# (Notez votre ID de VM. Exemple dans la vidéo : 00e7d220)

# Trouver les identifiants Vendor et Product de votre clé USB Zigbee :
lsusb
# (Notez votre ID de clé. Exemple dans la vidéo : 10c4:ea60)


# ==============================================================================
# ÉTAPE 2 : CRÉATION ET INJECTION DU FICHIER CONFIGURATION XML
# ==============================================================================

# Générer le fichier XML temporaire décrivant la clé USB (Modifiez 10c4 et ea60 si nécessaire) :
echo "<hostdev mode='subsystem' type='usb' managed='yes'>
  <source>
    <vendor id='0x10c4'/>
    <product id='0xea60'/>
  </source>
</hostdev>" | sudo tee /tmp/sonoff.xml

# Connecter immédiatement la clé USB à la machine virtuelle (Modifiez l'ID de la VM) :
sudo virsh attach-device 00e7d220 --file /tmp/sonoff.xml --current

# [Optionnel] Vérifier dans le terminal de Home Assistant que la clé est bien vue :
ha hardware info | grep -i "ttyUSB"


# ==============================================================================
# ÉTAPE 3 : SAUVEGARDE PERMANENTE ET SCRIPT D'AUTO-CONNEXION
# ==============================================================================

# Sauvegarder le fichier XML dans l'espace de stockage permanent de ZimaOS :
sudo cp /tmp/sonoff.xml /DATA/sonoff.xml

# Créer le script d'automatisation intelligent (Boucle d'attente active du démarrage de la VM) :
echo '#!/bin/bash
# On attend activement que la VM passe en mode "running"
while [ $(sudo virsh domstate 00e7d220 | grep -c "running") -eq 0 ]; do
  echo "Attente du démarrage de la VM..."
  sleep 5
done

# Petite pause de sécurité supplémentaire de 10 secondes
sleep 10

# On ré-injecte la clé USB à chaud dans la VM
sudo virsh attach-device 00e7d220 --file /DATA/sonoff.xml --current' | sudo tee /DATA/attach_sonoff.sh

# Rendre le script d'auto-connexion exécutable :
sudo chmod +x /DATA/attach_sonoff.sh


# ==============================================================================
# ÉTAPE 4 : PLANIFICATION AU DÉMARRAGE DU SERVEUR (CRONTAB)
# ==============================================================================

# Ouvrir le planificateur de tâches de ZimaOS :
sudo EDITOR=nano crontab -e

# >>> COLLER LA LIGNE SUIVANTE TOUT EN BAS DU FICHIER CRONTAB :
# @reboot /DATA/attach_sonoff.sh

# RAPPEL COMPORTEMENT NANO : 
# 1. Ctrl + O puis Entrée (Pour sauvegarder)
# 2. Ctrl + X (Pour quitter l'éditeur)
