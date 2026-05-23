# 🤖 Rendre un aspirateur Roborock accessible aux Seniors avec un bouton Zigbee

![Home Assistant](https://img.shields.io/badge/Home__Assistant-%2341BDF5.svg?style=for-the-badge&logo=home-assistant&logoColor=white)
![Zigbee](https://img.shields.io/badge/Zigbee-%23EB144C.svg?style=for-the-badge&logo=zigbee&logoColor=white)
![Roborock](https://img.shields.io/badge/Roborock-%23DF0024.svg?style=for-the-badge&logo=roborock&logoColor=white)

Ce dépôt contient les deux automatisations Home Assistant (YAML) permettant de piloter un aspirateur robot Roborock à l'aide d'un simple bouton physique Zigbee (testé avec un modèle Tuya TS004F sous ZHA).

---

## 🎯 Le Concept Fonctionnel

| Geste sur le bouton | Action du Roborock | Événement Zigbee (ZHA) |
| :--- | :--- | :--- |
| **1 clic normal** | Lance le nettoyage de la **Cuisine** | `toggle` |
| **2 clics successifs** | Arrête le robot et **Retour à la base** | `toggle` + `toggle` |
| **Double-clic rapide** | Arrête le robot et **Retour à la base** | `on` |

> [!TIP]
> **L'astuce logicielle :** Nous utilisons le mode `restart` d'Home Assistant pour mesurer électroniquement l'intervalle de temps entre deux clics. Cela permet d'adapter la réactivité du système au rythme physique de l'utilisateur (idéal pour les personnes âgées), sans forcer un double-clic trop rapide.

---

## 🛠️ Les Automatisations (YAML)

> [!IMPORTANT]
> Avant de copier les codes, pensez à adapter le `device_id` de votre bouton et l'entité `vacuum` de votre robot.

### 📦 1. Automatisation : Lancement du nettoyage (Cuisine)
<details>
<summary><b>▶ Cliquez ici pour dérouler le code YAML</b></summary>

```yaml
alias: "Bouton Roborock - Cuisine (Senior)"
description: "Lance la cuisine uniquement s'il n'y a pas un deuxième clic"
mode: single # "single" ignore les nouveaux clics tant que le délai de 1.5s n'est pas expiré
triggers:
  - trigger: event
    event_type: zha_event # Écoute les événements bruts du protocole Zigbee via ZHA
    event_data:
      device_id: 061c11084375077bf9f0d41c78a44faa # ⚠️ À remplacer par votre ID d'appareil
      command: "toggle" # Événement reçu lors d'un clic simple ou lent
conditions:
  - condition: state
    entity_id: vacuum.roborock_qv_35a # ⚠️ À remplacer par votre entité de robot
    state: "docked" # Sécurité : Le robot doit être sur sa base pour démarrer
actions:
  - delay: "00:00:01.500" # Temps d'attente pour laisser une fenêtre au double-clic d'arrêt
  - action: vacuum.clean_area
    target:
      entity_id: vacuum.roborock_qv_35a
    data:
      cleaning_area_id:
        - cuisine # Le nom de la pièce (doit être identique dans l'app Roborock et dans Zones HA)

### 📦 2. Automatisation : Arrêt d'urgence et Retour à la Base
<details>
<summary><b>▶ Cliquez ici pour dérouler le code YAML</b></summary>

```yaml
alias: "Bouton Roborock - Arrêt et Retour Base (Senior)"
description: "Retour base avec double-clic rapide (On) OU deux clics successifs (Toggle)"
mode: restart # Crucial : chaque clic redémarre le script à zéro et casse le délai précédent
triggers:
  - trigger: event
    event_type: zha_event
    event_data:
      device_id: 061c11084375077bf9f0d41c78a44faa # ⚠️ À remplacer par votre ID d'appareil
      command: "on" # Reçu lors d'un double-clic matériel ultra-rapide
  - trigger: event
    event_type: zha_event
    event_data:
      device_id: 061c11084375077bf9f0d41c78a44faa # ⚠️ À remplacer par votre ID d'appareil
      command: "toggle" # Reçu lors d'un clic normal
conditions:
  # CONDITION CLÉ : Bloque le script si le robot est déjà au dock (sauf s'il s'agit du double-clic rapide "on")
  - condition: template
    value_template: "{{ trigger.event.data.command == 'on' or states('vacuum.roborock_qv_35a') != 'docked' }}"
actions:
  # ÉTAPE 1 : Si le robot travaille, n'importe quel clic l'arrête immédiatement (Sécurité)
  - if:
      - condition: not
        conditions:
          - condition: state
            entity_id: vacuum.roborock_qv_35a
            state: "docked"
    then:
      - action: vacuum.stop
        target:
          entity_id: vacuum.roborock_qv_35a

  # ÉTAPE 2 : Analyse de la vitesse et du type de clics
  - choose:
      # Cas A : C'est le double-clic matériel rapide ("on"), on passe directement à la suite
      - conditions:
          - "{{ trigger.event.data.command == 'on' }}"
        sequence: [] 

      # Cas B : C'est un clic normal ("toggle"). On attend de voir si un deuxième arrive.
      - conditions:
          - "{{ trigger.event.data.command == 'toggle' }}"
        sequence:
          - delay: "00:00:01.800" # Fenêtre de temps accordée pour faire le deuxième clic
          # Si le délai s'écoule SANS deuxième clic, on bloque l'exécution ici (c'était un clic simple)
          - condition: template
            value_template: "{{ false }}"

  # ÉTAPE 3 : Action finale. Exécutée si l'attente a été brisée par un 2ème clic (mode restart)
  - action: vacuum.return_to_base
    target:
      entity_id: vacuum.roborock_qv_35a
