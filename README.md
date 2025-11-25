# 🛡️ Home Lab Réseau Avancé & Sécurité Opérationnelle

[![Statut du Projet](https://img.shields.io/badge/Statut-En%20Cours-orange)](./documentation/objectifs.md)
[![Technologies Principales](https://img.shields.io/badge/Tech-pfSense%2C%20Proxmox%2C%20Ansible-blue)](./documentation/architecture.md)
[![Focus Technique](https://img.shields.io/badge/Focus-R%C3%A9seau%20Avanc%C3%A9%20%26%20S%C3%A9curit%C3%A9-red)](./documentation/rapport_technique.md)

Ce dépôt documente le déploiement d'un Home Lab réseau complexe, virtualisé sur des hôtes **Proxmox VE**, simulant une infrastructure d'entreprise hautement segmentée. Le projet met en évidence la maîtrise du **routage sécurisé (pfSense)**, la gestion des accès distants (**VPN**), l'**automatisation (Ansible)**, et l'**audit technique** via des outils professionnels de **Documentation d'Infrastructure (NetBox)** et de **Monitoring (LibreNMS/Grafana)**.

---

## 🎯 Objectifs du Projet

Ce laboratoire est conçu pour valider une **maîtrise complète des architectures réseaux modernes, de la sécurité opérationnelle et des pratiques d'audit technique**.

* **Routage & Segmentation :** Configurer **pfSense A** comme firewall/routeur inter-VLAN principal pour appliquer des politiques de sécurité strictes, assurant le **principe du moindre privilège**.
* **Virtualisation & Distribution :** Utiliser des conteneurs **LXC** et des **VMs** distribués sur deux hôtes Proxmox (PC A et PC B) pour optimiser les ressources.
* **Contrôle et Visibilité :** Déployer une stack de monitoring professionnelle (**LibreNMS, Grafana, ntopng**) pour la surveillance proactive du réseau et l'analyse des flux.
* **Audit et Documentation d'Infrastructure :** Mettre en place **NetBox** pour l'**IPAM** (Gestion des Adresses IP) et l'inventaire, et **Oxidized** pour la sauvegarde automatisée des configurations, des étapes clés de l'audit et de la traçabilité.
* **Automatisation :** Utiliser **Ansible** pour le déploiement rapide et reproductible des services (IaC - Infrastructure as Code).

<p align="center">
<img width="650" src="https://github.com/tescalon/Advanced-Network-Lab/blob/main/Architecture%20r%C3%A9elle%20r%C3%A9seau.png" alt="Architecture réelle" />
</p>

---
**Le projet est actuellement en cours de réalisation. La documentation détaillée de l'architecture, de la configuration et des analyses de sécurité sera mise à jour et publiée dans les prochains jours / semaines / mois.**
