# 🛡️ Home Lab Réseau & Sécurité

[![Statut du Projet](https://img.shields.io/badge/Statut-Finalis%C3%A9-success)](./docs/ARCHITECTURE.md)
[![Focus Technique](https://img.shields.io/badge/Focus-Réseaux%20%7C%20Sécurité&GRC%20%7C%20Zero%20Trust%20%7C%20IaC-blue)](./docs/ARCHITECTURE.md)
[![Infra](https://img.shields.io/badge/Infra-pfSense%2C%20Proxmox%2C%20WireGuard%2C%20Docker-critical)](./docs/ARCHITECTURE.md)
[![Ops Stack](https://img.shields.io/badge/Ops-Ansible%2C%20NetBox%2C%20LibreNMS%2C%20Grafana%2C%20ntopng-blueviolet)](./DOCKER_STACK/docker-compose.yml)

> **Projet Académique & Personnel** - Simulation d'une infrastructure d'entreprise segmentée avec contraintes de conformité. Ce dépôt documente le déploiement d'une **infrastructure multisite** (Siège/Agence) simulant un environnement critique, orientée **Sécurité Réseau & GRC**.

---

## 📑 Table des Matières (Navigation Rapide)

1.  [Piliers Architecturaux et Sécurité](#1--piliers-architecturaux-et-sécurité)
2.  [Isolation L2 : "Physical Virtual Segregation"](#2-isolation-l2--physical-virtual-segregation)
3.  [Architecture & Inventaire IPAM](#3%EF%B8%8F-architecture--inventaire-ipam)
4.  [Ingénierie & Durcissement](#4-%EF%B8%8F-ingénierie--durcissement)
5.  [Stack GRC et Automatisation](#%EF%B8%8F-5-stack-technique-réseau--sécurité--grc)
6.  [Interconnexion Sécurisée (WireGuard)](#6--interconnexion-sécurisée-wireguard)
7.  [Politique de Sécurité (Zero Trust)](#7-%EF%B8%8F-politique-de-sécurité-zero-trust)
8.  [Aperçu Visuel & Preuves de Concept](#8--aperçu-visuel--preuves-de-concept)
9.  [Roadmap & Perspectives d'Évolution](#9-%EF%B8%8F-roadmap--perspectives-dévolution)
10. [Compétences Démontrées](#10--compétences-démontrées)

---

## 1. 🏢 Piliers Architecturaux et Sécurité

Le projet dépasse la simple connectivité pour simuler un environnement critique où chaque flux est justifié. L'approche est celle du **"Security by Design"** : l'architecture privilégie une segmentation stricte et une auditabilité totale.

**Piliers de l'architecture :**
* **Isolation Réseau (L2) :** Stratégie de "Physical Virtual Segregation" via des interfaces vNICs distinctes pour neutraliser les risques de *VLAN Hopping*.
* **Virtualisation & Conteneurisation :** Orchestration sous **Proxmox** avec une stack applicative **Docker** encapsulée dans des **LXC non-privilégiés**, garantissant une isolation kernel stricte entre les services critiques.
* **Connectivité Furtive :** Tunneling **WireGuard** Site-à-Site optimisé pour la furtivité (port UDP 51820 invisible aux scans non-authentifiés).
* **Visibilité & Conformité :** Stratégie de supervision hybride (Edge avec *ntopng* / Central avec *LibreNMS*) pilotée par une "Source of Truth" unique (**NetBox**).
* **Infrastructure as Code :** Audits de conformité automatisés via **Ansible**, assurant qu'aucun changement manuel ne passe inaperçu (Anti-Drift).

---

## 2.🔌 Isolation L2 : "Physical Virtual Segregation"

Cette architecture répond à une problématique spécifique liée à la sécurité des environnements virtualisés imbriqués (*Nested Virtualization*).

> **⚠️ Le Risque Identifié (Threat Model)**
> Dans les environnements virtuels, la gestion des tags VLAN (**802.1Q**) peut être aléatoire (phénomène de *VLAN Stripping*), introduisant un risque majeur de **VLAN Hopping**. Un attaquant pourrait théoriquement "sauter" d'une zone compromise (DMZ) vers une zone sûre (LAN) sans passer par le filtrage du pare-feu.

### La Solution : "Air Gap Virtuel"

Au lieu de faire passer tous les réseaux sur un seul câble virtuel (Mode Trunk), nous appliquons une **isolation stricte par interface**.
* **Approche Classique (Rejetée) :** 1 vNIC avec Trunk VLAN $\rightarrow$ Risque de fuite.
* **Approche Retenue (Ségrégation) :** 1 vNIC distincte connectée à un Pont Linux (Bridge) distinct pour chaque zone.

---

## 3.🏗️ Architecture & Inventaire IPAM

Le cœur du réseau est hébergé sur le site principal. Il concentre les fonctions de sécurité périmétrique et de gouvernance.

### 3.1. Schéma d'Architecture 

[Image of Network Topology Diagram showing HQ LAN, DMZ, Branch LAN, and VPN tunnel connecting them, with IP subnets and pfSense routers]

[**Voir le Fichier Complet de l'Architecture et des Configurations dans `docs/ARCHITECTURE.md`**](./docs/ARCHITECTURE.md)

### 3.2. Plan d'Adressage (IPAM)
L'adressage utilise la RFC1918 et une logique géographique stricte.

| Zone | CIDR (L3) | Gateway (pfSense) | Élément Clé & IP |
| :--- | :--- | :--- | :--- |
| **LAN Siège** | `10.10.10.0/24` | `10.10.10.254` | **Hyperviseur pve (Proxmox):** `10.10.10.15` |
| **DMZ Siège** | `10.50.10.0/24` | `10.50.10.254` | **Serveur Admin/Docker:** `10.50.10.10` |
| **LAN Agence** | `10.20.10.0/24` | `10.20.10.254` | **Client Agence (Debian):** `10.20.10.10` |
| **VPN** | `10.10.20.0/24` | - | **WireGuard Peer HQ:** `.1` / **Peer BR:** `.2` |

---

## 4. 🛡️ Ingénierie & Durcissement

Cette section détaille les choix techniques effectués pour renforcer la sécurité et la stabilité du système.

### 4.1. Configuration pfSense (Cœur de Réseau)
*Rôle : Security Gateway & Point de terminaison VPN.*

#### Interfaces & Ségrégation
Chaque interface correspond à une zone de sécurité isolée physiquement (vNIC distincte).

| Interface | Zone | IP / CIDR | Rôle & Politique de Sécurité |
| :--- | :--- | :--- | :--- |
| **WAN** (`em0`) | *Untrusted* | `DHCP / Public` | Connecté au monde extérieur. Règle **"Deny All"** en entrée par défaut. |
| **LAN** (`em1`) | *Trust* | `10.10.10.254/24` | Zone de Gestion. Accès administrateur complet. |
| **SECOPS** (`em2`) | *DMZ* | `10.50.10.254/24` | **Zone Démilitarisée.** Isolation stricte (Pas d'accès initié vers le LAN). |
| **VPN** (`em3`) | *Overlay* | `10.10.20.1/24` | Interface virtuelle **WireGuard**. Transport chiffré inter-sites. |

#### Optimisation Kernel (Intégrité des Données)
> **Configuration Critique : Hardware Checksum Offload = DISABLED**
>
> * **Justification Technique :** Les drivers paravirtualisés (**VirtIO**) calculent parfois mal les sommes de contrôle (Checksums) TCP/UDP.
> * **Impact évité :** Empêche la corruption silencieuse des paquets et l'apparition de faux positifs sur les systèmes de détection d'intrusion (IDS).

#### Services Réseau & Résilience
* **DNS Resolver (Unbound) :** Mode récursif avec *Host Overrides* pour le domaine interne `netbox.homelab`. *(Gain GRC : Évite la dépendance aux DNS publics et masque la topologie interne (Privacy)).*
* **Auto Config Backup (ACB) :** Sauvegarde chiffrée (**AES-256**) automatique dans le cloud pfSense. *(Gain GRC : Garantit un **RTO (Recovery Time Objective)** minimal en cas de crash matériel).*

### 4.2. Serveur d'Administration (`srv-admin-siege: 10.50.10.10`)
*Type : Conteneur LXC (ID 105)*

#### Architecture : "Docker on LXC"
L'architecture utilise une imbrication de conteneurs (Nesting) pour optimiser les ressources sans sacrifier la sécurité.
* **Justification Hardening (Durcissement) :**
    * **LXC Non-Privilégié (Unprivileged) :** Le `root` du conteneur est mappé sur un utilisateur standard de l'hôte.
    * **Option `nesting=1` :** Permet l'isolation des namespaces Docker.

[**Voir la Configuration du Conteneur LXC (`105.conf`) dans `docs/ARCHITECTURE.md`**](./docs/ARCHITECTURE.md)

---

## 🛠️ 5. Stack Technique Réseau & Sécurité & GRC

La chaîne d'outillage est centralisée dans la DMZ pour respecter la **Ségrégation des Tâches (SoD)**.

### 5.1. Stack Applicative GRC

| Service | Rôle GRC | Justification du Choix |
| :--- | :--- | :--- |
| **NetBox** | *Source of Truth* | **Asset Management.** Remplace les fichiers Excel obsolètes. Documente chaque câble, IP et VLAN *avant* déploiement. |
| **LibreNMS** | *Supervision* | Utilisation exclusive de **SNMPv3** (Authentifié & Chiffré) indispensable pour traverser des zones non sûres. |
| **Grafana** | *Visualisation* | Centralisation des KPIs de disponibilité pour les tableaux de bord directionnels. |
| **Oxidized** | *Sauvegarde* | **Traçabilité & Audit.** Versioning automatique des configs. Répond à la question *"Qui a changé quoi et quand ?"* (Diff configs). |

[**Voir le fichier `docker-compose.yml` complet dans `DOCKER_STACK/`**](./DOCKER_STACK/docker-compose.yml)

### 5.2. Automatisation (Ansible)
* **Objectif :** Éliminer la configuration manuelle et le risque de *Configuration Drift*.
* **Sécurité :** Utilisation de clés SSH **Ed25519** (Courbes elliptiques, plus robuste que RSA) pour l'authentification sans mot de passe vers les pare-feux.

[**Voir les Playbooks d'Audit et de Durcissement dans `ANSIBLE/`**](./ANSIBLE/)

### 5.3. Infrastructure Agence (Surveillance "Edge")
Nous adoptons une stratégie de traitement à la périphérie (**Edge Computing**) pour éviter de saturer le lien VPN avec du trafic de monitoring brut.
* **Composant :** `ntopng` installé directement sur `pfsense-agence`.
* **Rôle :** Analyseur de flux (NetFlow/IPFIX). *(Justification GRC : Permet une **détection d'anomalies locales** en temps réel, sans impact sur la bande passante inter-sites.)*

| Logo | Nom | Rôle |
| :---: | :--- | :--- |
| <img src="https://github.com/tescalon/Homelab-Network-Secops/blob/main/docs/images/logo/ansible.png?raw=true" width="60"> | **Ansible** | **Infrastructure as Code (IaC) & Audit.** Automatisation des déploiements et vérification de la **Conformité** (Anti-Drift). |
| <img src="https://github.com/tescalon/Homelab-Network-Secops/blob/main/docs/images/logo/pfsense.png?raw=true" width="50"> | **pfSense** | **Sécurité Périmétrique.** Firewalling, Routage inter-zones, Terminaux VPN. Cœur de la **Défense en Profondeur**. |
| <img src="https://github.com/tescalon/Homelab-Network-Secops/blob/main/docs/images/logo/proxmox.png?raw=true" width="60"> | **Proxmox VE** | **Virtualisation & Isolation L2.** Hyperviseur Type 1 garantissant l'isolation physique virtuelle (`vmbrX`). |
| <img src="https://github.com/tescalon/Homelab-Network-Secops/blob/main/docs/images/logo/docker.png?raw=true" width="60"> | **Docker** | **Conteneurisation sécurisée.** Orchestration de la stack GRC dans des LXC non-privilégiés. |
| <img src="https://github.com/tescalon/Homelab-Network-Secops/blob/main/docs/images/logo/wireguard.png?raw=true" width="60"> | **WireGuard** | **Tunneling Sécurisé.** VPN Site-à-Site pour la **Confidentialité** et l'intégrité des données inter-sites. |
| <img src="https://github.com/tescalon/Homelab-Network-Secops/blob/main/docs/images/logo/netbox.png?raw=true" width="60"> | **NetBox** | **Gouvernance & CMDB.** Unique **Source of Truth** (SoT) pour l'IPAM et l'inventaire des actifs (GRC Data Quality). |
| <img src="https://github.com/tescalon/Homelab-Network-Secops/blob/main/docs/images/logo/librenms.png?raw=true" width="60"> | **LibreNMS** | **Supervision & Alerting.** Collecte des métriques via **SNMPv3** (chiffré) pour la **Disponibilité** et la sécurité des données de monitoring. |
| <img src="https://github.com/tescalon/Homelab-Network-Secops/blob/main/docs/images/logo/grafana.png?raw=true" width="60"> | **Grafana** | **Visualisation & Observabilité.** Tableaux de bord *Single Pane of Glass* centralisant les KPIs (LibreNMS, ntopng). |
| <img src="https://img.shields.io/badge/Oxidized-17202A?style=flat&logo=git&logoColor=white" width="60"> | **Oxidized** | **Audit & Traçabilité.** Versioning automatique des configurations routeurs pour la **Traçabilité des Changements** (GitOps). |
| <img src="https://github.com/tescalon/Homelab-Network-Secops/blob/main/docs/images/logo/ntopng.png?raw=true" width="60"> | **ntopng** | **Analyse de Flux (Edge).** Détection d'anomalies et analyse comportementale du trafic à la périphérie (Agence). |
---

## 6. 🔒 Interconnexion Sécurisée (WireGuard)

Choix technologique : **WireGuard** (vs IPsec/OpenVPN).

### 6.1. Justification Cryptographique & Performance
* **Surface d'attaque réduite :** ~4 000 lignes de code (facilitant l'audit de sécurité).
* **Cryptographie Moderne :** Utilise **ChaCha20-Poly1305** et **Curve25519**.
* **Stealth (Furtivité) :** WireGuard ne répond pas aux paquets non authentifiés. Pour un scanner externe, le port UDP `51820` apparaît **fermé** ou invisible.

### 6.2. Architecture de Routage (Statique)
* **Choix :** Routage Statique.
* **Justification :** Évite les risques d'**injection de routes malveillantes**. Le trafic suit strictement le chemin défini en dur.

---

## 7. 🛡️ Politique de Sécurité (Zero Trust)

**Stratégie appliquée :** Zero Trust (Default Deny). Le pare-feu est configuré pour bloquer par défaut tout trafic non explicitement autorisé.

| Interface | Source | Destination | Port / Proto | Action | Commentaire / Justification GRC |
| :--- | :--- | :--- | :--- | :---: | :--- |
| **WAN** | *Any* | WAN Address | `UDP/51820` | **✅ Pass** | Établissement du Tunnel WireGuard. |
| **LAN Siège** | LAN Net | *Any* | *Any* | **✅ Pass** | Zone de Gestion de Confiance (Trust). |
| **DMZ** | DMZ Net | RFC1918 (LANs) | *Any* | **❌ Block** | **Isolation Critique.** La DMZ ne peut jamais initier de connexion vers le LAN Admin. |
| **DMZ** | DMZ Net | *Any* (Internet) | *Any* | **✅ Pass** | Accès sortant uniquement (Mises à jour / Repositories). |
| **VPN** | Agence Net | DMZ Net | `TCP/80, 3000, 8000` | **✅ Pass** | Accès aux outils GRC (NetBox, Grafana) depuis l'agence. |
| **VPN** | Siège Net | Agence Net | `UDP/161` | **✅ Pass** | Flux de supervision (Pull SNMP) vers l'agence. |

[**Voir la Matrice de Flux détaillée par Interface (WAN/LAN/DMZ) dans `docs/FIREWALL_RULES.md`**](./docs/FIREWALL_RULES.md)

---

## 8. 📸 Aperçu Visuel & Preuves de Concept

Afin de valider l'architecture et les politiques de sécurité clés, voici cinq preuves concrètes de la mise en œuvre de notre Lab.

#### 1. Preuve Critique du Zero Trust (Isolation L3)

Ceci est la **preuve fondamentale de sécurité**. Elle montre la règle de pare-feu qui **bloque** toute tentative de connexion initiée depuis la DMZ vers le réseau d'administration (LAN HQ).

> **Justification :** Sans cette règle, un attaquant ayant compromis la Stack Docker pourrait se déplacer vers le réseau de gestion.

![Règle BLOCK Critique (DMZ)](docs/images/Host_A/pfsense_zero_trust_block.png)

#### 2. Validation du Routage Inter-Sites (WireGuard S2S)

Le `traceroute` prouve que la connectivité entre le Siège et l'Agence est fonctionnelle et qu'elle passe correctement par la Gateway du tunnel VPN (`10.10.20.2`).

> **Justification :** Valide le tunnel VPN WireGuard et prouve la correction du routage statique complexe (Ingénierie Réseau).

![Traceroute S2S](docs/images/Host_A/validation_traceroute_vpn.png)

#### 3. Hardening et Optimisation Kernel

Preuve d'une correction essentielle en environnement virtualisé : la désactivation de l'Offloading pour garantir l'intégrité des données TCP/UDP et éviter la corruption de paquets.

> **Justification :** Démontre l'expertise dans la gestion des problèmes de performance et d'intégrité de niveau kernel.

![Optimisation Kernel](docs/images/Host_A/pfsense_kernel_optimisation.png)

#### 4. Audit, Supervision et Persistance (GRC)

Preuve que l'ensemble des outils SecOps (NetBox, LibreNMS, Oxidized) est déployé et opérationnel via Docker Compose.

> **Justification :** Valide l'orchestration, l'auditabilité et la disponibilité des services critiques GRC.

![Statut Docker Stack](docs/images/Host_A/docker_stack_status.png)

#### 5. Organisation IaC pour la Conformité

La structure du répertoire du serveur d'administration prouve l'organisation des fichiers pour le déploiement et l'audit (Ansible, Docker, Oxidized).

> **Justification :** Démontre la rigueur dans l'application des principes d'Infrastructure as Code (IaC) et GitOps.

![Organisation IaC](docs/images/Host_A/homelab_file_structure.png)

---

> **Pour une analyse complète de l'architecture, de la Matrice de Flux GRC et des 34 preuves visuelles, veuillez consulter tous les documents et les images dans le dossier `docs/`.**

---

## 9. ⚙️ Roadmap & Perspectives d'Évolution

Ce plan d'action définit les évolutions futures pour maintenir le niveau de sécurité, de conformité et de performance de l'infrastructure.

| Phase | Tâche | Justification GRC / Exploitation |
| :--- | :--- | :--- |
| **I. Sécurité** | **Durcissement SSH (Hardening)** | Désactivation totale de l'auth par mot de passe sur pfSense une fois les clés Ed25519 déployées via Ansible (Mitigation Brute-force). |
| **I. Sécurité** | **Accès Zero Trust (Cloudflare Tunnel)** | Mise en place du Tunnel Cloudflare pour l'accès aux outils GRC/Ops (NetBox/Grafana). Élimine l'exposition publique des ports et fournit une couche d'authentification forte. |
| **II. Audit** | **Audit de Conformité Automatisé** | Finalisation du playbook Ansible vérifiant périodiquement l'état des configurations par rapport au référentiel ("Configuration Drift"). |
| **III. Visibilité** | **Intégration Single Pane of Glass** | Injection des données de flux **ntopng** dans les dashboards **Grafana** pour corréler métriques systèmes et comportement réseau. |
| **IV. Data Quality** | **Fiabilisation CMDB (NetBox)** | Peupler 100% des objets pour que NetBox devienne l'unique "Source of Truth" opposable. |
| **V. Alerting** | **Alerting Critique** | Configuration des seuils d'alerte LibreNMS (ex: *VPN Down*, *Disk Usage > 80%*) avec notifications. |
| **VI. Sauvegarde** | **GitOps Réseau (Oxidized)** | Automatisation complète du versioning des configurations routeurs vers un dépôt Git (Traçabilité des changements). |
| **VII. SDN** | **Proxmox SDN (VXLAN)** | Migration des Linux Bridges vers une architecture **Software Defined Network** (VXLAN) pour une segmentation indépendante de l'infrastructure physique. |
| **VIII. Access Control** | **NAC 802.1X (RADIUS)** | Implémentation du contrôle d'accès réseau : aucun port ne s'active sans authentification du périphérique via certificats (Zero Trust au niveau Layer 2). |
| **IX. Résilience** | **Haute Disponibilité (CARP)** | Configuration d'un cluster pfSense actif/passif avec synchronisation d'état (pfsync) pour garantir la continuité de service en cas de panne matérielle (Business Continuity Plan). |

> **L'implémentation des tâches de la Roadmap (Sécurité, Audit, Résilience) est planifiée pour les prochains jours ou semaines**
---

## 10. ✅ Compétences Démontrées

Ce projet met en œuvre des compétences transversales en ingénierie système et sécurité.

### Cybersécurité & Hardening
* **Défense en Profondeur :** Conception d'une architecture cloisonnée (DMZ, LAN, Management) avec ségrégation stricte au niveau 2 (vNICs distinctes).
* **Stratégie Zero Trust :** Application de politiques de pare-feu "Default Deny" et restriction des flux inter-VLAN.
* **VPN & Cryptographie :** Déploiement de tunnels **WireGuard** site-à-site (Configuration des clés, routage statique).
* **Accès Distant Sécurisé :** Mise en place d'un tunnel **Cloudflare Zero Trust** pour l'administration sans exposition de surface d'attaque (No Open Ports).

### Architecture & Réseau (NetOps)
* **Gouvernance des Données (GRC) :** Utilisation de **NetBox** comme *Source of Truth* (SoT) pour piloter l'inventaire.
* **Supervision Hybride :** Implémentation d'une stratégie de monitoring centralisée (**LibreNMS/SNMPv3**) couplée à une analyse de flux déportée en "Edge" (**ntopng**).
* **Virtualisation Avancée :** Maîtrise de l'hyperviseur **Proxmox VE** (Gestion des ponts Linux, conteneurs LXC non-privilégiés, nesting Docker).

### Automatisation & Audit (DevSecOps)
* **Infrastructure as Code (IaC) :** Utilisation d'**Ansible** pour le déploiement standardisé des configurations et le durcissement des accès (Clés SSH).
* **Audit & Traçabilité :** Mise en place d'**Oxidized** pour le versioning automatique des configurations réseau (Détection de *Configuration Drift*).
* **Conteneurisation :** Orchestration de stacks applicatives via **Docker Compose** dans des environnements contraints.
