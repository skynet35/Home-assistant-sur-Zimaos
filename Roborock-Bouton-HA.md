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

### 📦 Codes YAML des Automatisations (Cuisine & Retour Base)
<details>
<summary><b>▶ Cliquez ici pour dérouler les codes des automatisations</b></summary>

```yaml
# ==============================================================================
# AUTOMATISATION 1 : LANCEMENT DU NETTOYAGE (CUISINE)
# Action : 1 clic normal ("toggle") -> attend 1.5s -> lance le robot
# ==============================================================================
alias: "Bouton Roborock - Cuisine (Senior)"
description: "Lance la cuisine uniquement s'il n'y a pas un deuxième clic"
mode: single
triggers:
  - trigger: event
    event_type: zha_event
    event_data:
      device_id: 061c11084784077bf9f9641c78a75faa # ⚠️ À remplacer par votre ID d'appareil
      command: "toggle"
conditions:
  - condition: state
    entity_id: vacuum.roborock_qv_35a # ⚠️ À remplacer par votre entité de robot
    state: "docked"
actions:
  - delay: "00:00:01.500"
  - action: vacuum.clean_area
    target:
      entity_id: vacuum.roborock_qv_35a
    data:
      cleaning_area_id:
        - cuisine


# ==============================================================================
# AUTOMATISATION 2 : ARRÊT D'URGENCE ET RETOUR A LA BASE
# Action : 2 clics successifs (ou double-clic rapide) -> renvoie le robot au dock
# ==============================================================================
alias: "Bouton Roborock - Arrêt et Retour Base (Senior)"
description: "Retour base avec double-clic rapide (On) OU deux clics successifs (Toggle)"
mode: restart
triggers:
  - trigger: event
    event_type: zha_event
    event_data:
      device_id: 061c11084784077bf9f9641c78a75faa # ⚠️ À remplacer par votre ID d'appareil
      command: "on"
  - trigger: event
    event_type: zha_event
    event_data:
      device_id: 061c11084784077bf9f9641c78a75faa # ⚠️ À remplacer par votre ID d'appareil
      command: "toggle"
conditions:
  - condition: template
    value_template: "{{ trigger.event.data.command == 'on' or states('vacuum.roborock_qv_35a') != 'docked' }}"
actions:
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
  - choose:
      - conditions:
          - "{{ trigger.event.data.command == 'on' }}"
        sequence: [] 
      - conditions:
          - "{{ trigger.event.data.command == 'toggle' }}"
        sequence:
          - delay: "00:00:01.800"
          - condition: template
            value_template: "{{ false }}"
  - action: vacuum.return_to_base
    target:
      entity_id: vacuum.roborock_qv_35a
