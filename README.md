# 🛡️ Home Lab Réseau Avancé & Sécurité Opérationnelle

[![Statut du Projet](https://img.shields.io/badge/Statut-En%20Cours-orange)](./documentation/objectifs.md)
[![Technologies Principales](https://img.shields.io/badge/Tech-pfSense%2C%20Proxmox%2C%20Ansible-blue)](./documentation/architecture.md)
[![Focus Technique](https://img.shields.io/badge/Focus-R%C3%A9seau%20Avanc%C3%A9%20%26%20S%C3%A9curit%C3%A9-red)](./documentation/rapport_technique.md)


> **Projet Académique & Personnel** - Simulation d'une infrastructure d'entreprise hautement segmentée avec contraintes de conformité.

## 📋 Présentation

Ce dépôt documente le déploiement d'un laboratoire virtuel simulant un réseau d'entreprise multisite. Contrairement aux "Labs" classiques, celui-ci est orienté **Gouvernance et Sécurité (GRC)** : chaque choix technique répond à une exigence de traçabilité, de moindre privilège ou de disponibilité.

**Points clés :**
* **Zero Trust Access :** Aucun port ouvert, accès via Cloudflare Tunnels authentifiés.
* **Infrastructure as Code :** Audit de conformité automatisé via Ansible.
* **Source of Truth :** Documentation réseau dynamique (NetBox) faisant foi.

---

## 📸 Aperçu Visuel (Screenshots)

### 1. Topologie Réseau Logique
*Générée via NetBox, illustrant la segmentation VLAN (Infra, SecOps, Transit).*
![Schéma Réseau](docs/images/network_topology.png)

### 2. Supervision & Métrologie
*Monitoring actif des interfaces critiques via LibreNMS (SNMPv3).*
![LibreNMS Dashboard](docs/images/librenms_graph.png)

### 3. Sécurité & Routage
*Règles de filtrage strictes sur pfSense (Inter-VLAN).*
![Règles Firewall](docs/images/pfsense_rules.png)

---

## 🏗️ Architecture Technique

| Couche | Technologie | Rôle |
| :--- | :--- | :--- |
| **Virtualisation** | Proxmox VE | Hyperviseur Type 2 (Linux Bridge & VLAN Aware) |
| **Réseau** | pfSense | Routage, Firewalling, DHCP |
| **IAM / Accès** | Cloudflare Zero Trust | Portail d'accès sécurisé (IdP) |
| **Automation** | Ansible | Déploiement de configs & Audit de conformité |
| **IPAM** | NetBox | Gestionnaire d'adresses IP et Inventaire |

*Pour les détails techniques complets (Plan d'adressage IP, VLANs), voir la [Documentation Architecture](docs/ARCHITECTURE.md).*

---

## 🚀 Déploiement & Automatisation

L'infrastructure utilise **Ansible** pour garantir la conformité des configurations.

**Exemple de Playbook d'Audit (GRC) :**
Ce script ne configure pas, il vérifie que les politiques de sécurité sont appliquées (ex: Firewall local actif).

```yaml
- name: Audit de Conformité
  tasks:
    - name: Check UFW Status
      command: ufw status
      register: ufw_status
      failed_when: "'inactive' in ufw_status.stdout"
