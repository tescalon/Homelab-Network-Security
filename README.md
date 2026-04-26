# Home Lab — Réseau & Sécurité

[![Statut](https://img.shields.io/badge/Statut-En%20cours-orange)](./docs/ARCHITECTURE.md)
[![Stack](https://img.shields.io/badge/Infra-pfSense%2C%20Proxmox%2C%20WireGuard%2C%20Docker-critical)](./docs/ARCHITECTURE.md)
[![Outils](https://img.shields.io/badge/Outils-Ansible%2C%20NetBox%2C%20LibreNMS%2C%20Oxidized%2C%20ntopng-blueviolet)](./docs/ARCHITECTURE.md)

Projet personnel monté dans le cadre de ma formation en Administration Systèmes et Réseaux. L'idée de départ était simple : reproduire une infrastructure d'entreprise chez moi, avec une vraie segmentation réseau, un VPN site-à-site fonctionnel, et des outils de supervision dignes de ce nom — le tout sans matériel dédié.

L'infra tourne sur deux machines physiques sous VMware Workstation. Le site "Siège" héberge un Proxmox en nested virtualisation avec pfSense, un LXC Docker pour les outils de monitoring, et un tunnel WireGuard vers le site "Agence" simulé sur la deuxième machine.

---

## Table des Matières

1. [Environnement & Architecture](#1-environnement--architecture)
2. [Pourquoi une vNIC par zone ?](#2-pourquoi-une-vnic-par-zone-)
3. [Plan d'adressage](#3-plan-dadressage)
4. [pfSense — Configuration](#4-pfsense--configuration)
5. [Stack de supervision (Docker)](#5-stack-de-supervision-docker)
6. [VPN WireGuard site-à-site](#6-vpn-wireguard-site-à-site)
7. [Règles de filtrage](#7-règles-de-filtrage)
8. [Screenshots & preuves de fonctionnement](#8-screenshots--preuves-de-fonctionnement)
9. [Ce qui reste à faire](#9-ce-qui-reste-à-faire)
10. [Ce que ce projet m'a appris](#10-ce-que-ce-projet-ma-appris)

---

## 1. Environnement & Architecture

Tout tourne sur deux machines personnelles, pas sur du vrai matériel réseau.

**Machine principale (Siège) :** VMware Workstation avec Proxmox VE en nested virtualisation. Les VMnets VMware en mode Host-only assurent l'isolation L2 entre les zones — les Linux Bridges Proxmox (`vmbr0`, `vmbr1`) s'appuient directement dessus. À l'intérieur tourne pfSense Siège et un LXC non-privilégié qui héberge la stack Docker.

**Machine secondaire (Agence) :** VMware Workstation avec pfSense Agence directement, plus un client Debian simulant un poste utilisateur.

C'est du homelab, donc il y a des compromis — mais les concepts démontrés (segmentation, VPN, supervision, filtrage) sont réels et fonctionnels.

[→ Architecture détaillée dans `docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md)

---

## 2. Pourquoi une vNIC par zone ?

En environnement virtualisé, faire passer plusieurs VLANs sur une seule interface (mode trunk) peut poser un problème : certains hyperviseurs gèrent mal les tags 802.1Q, ce qui peut créer des fuites entre zones (VLAN hopping). Un service compromis en DMZ pourrait théoriquement atteindre le LAN admin sans passer par le pare-feu.

Pour éviter ça, j'ai choisi de dédier une interface physique virtuelle (vNIC) à chaque zone réseau plutôt que de faire du trunk. Chaque bridge Proxmox est isolé des autres au niveau VMware — c'est plus rigide à configurer mais bien plus sûr.

- **Approche trunk (abandonnée) :** 1 vNIC + tags VLAN → risque de fuite entre zones
- **Approche retenue :** 1 vNIC dédiée par zone → isolation garantie au niveau hyperviseur

---

## 3. Plan d'adressage

J'ai utilisé la logique `10.SITE.x.x` pour que les IPs soient lisibles sans avoir à consulter un document à chaque fois.

| Zone | Réseau | Gateway | Machines |
| :--- | :--- | :--- | :--- |
| LAN Siège | `10.10.10.0/24` | `10.10.10.254` | Proxmox : `10.10.10.15` |
| DMZ Siège | `10.50.10.0/24` | `10.50.10.254` | srv-admin-hq (Docker) : `10.50.10.10` |
| LAN Agence | `10.20.10.0/24` | `10.20.10.254` | Client Debian : `10.20.10.130` |
| Tunnel VPN | `10.10.20.0/24` | — | Peer Siège : `.1` / Peer Agence : `.2` |

---

## 4. pfSense — Configuration

pfSense est le cœur du réseau sur les deux sites. Il gère le routage inter-zones, le filtrage, la terminaison WireGuard et les services réseau de base.

### Interfaces (Siège)

| Interface | IP | Zone | Rôle |
| :--- | :--- | :--- | :--- |
| `em0` (WAN) | DHCP | Non fiable | Accès Internet simulé via NAT VMware |
| `em1` (LAN) | `10.10.10.254/24` | Admin | Gestion, accès complet |
| `em2` (SECOPS_DMZ) | `10.50.10.254/24` | DMZ | Services Docker, monitoring |
| `tun_wg0` (VPN) | `10.10.20.1/24` | Overlay | Tunnel WireGuard vers l'Agence |

### Optimisations kernel

En environnement virtualisé imbriqué, j'ai désactivé le TCP Segmentation Offload (TSO) et le Large Receive Offload (LRO) — ces deux options peuvent causer des corruptions de paquets avec certains drivers paravirtualisés. Le Hardware Checksum Offload reste actif, pas problématique avec VirtIO dans cette config.

### Autres services

- **DNS (Unbound)** en mode récursif avec host overrides pour `librenms.homelab.lan` et `netbox.homelab.lan` → résolution des noms internes sans dépendre d'un DNS externe
- **Auto Config Backup** activé — sauvegarde chiffrée AES-256 automatique de la config pfSense
- **SNMP** actif sur LAN et VPN uniquement, community string dédiée. Le trafic passe dans le tunnel WireGuard donc chiffré en transit

---

## 5. Stack de supervision (Docker)

Tous les outils de monitoring tournent en conteneurs Docker dans un LXC non-privilégié sur Proxmox. Le LXC non-privilégié mappe le `root` du conteneur sur un utilisateur sans droits sur l'hôte — si quelque chose est compromis, l'impact est limité.

| Service | Rôle | Port |
| :--- | :--- | :--- |
| **NetBox** | Inventaire réseau (IPAM, CMDB) | `8000` |
| **LibreNMS** | Supervision SNMP, graphes, alerting | `80` |
| **Oxidized** | Backup automatique des configs routeurs avec versioning Git | `8888` |
| **Cloudflared** | Tunnel Cloudflare Zero Trust — accès distant sans port public ouvert | — |

Grafana est prévu pour centraliser les métriques LibreNMS — il est dans le `docker-compose.yml` mais commenté pour l'instant, le temps de finaliser la source de données.

**ntopng** est installé différemment : c'est un package natif pfSense sur le site Agence, pas dans Docker. Ça lui permet d'analyser tout le trafic directement sur le firewall avant le routage — bien plus pertinent que de le déporter côté Siège.

[→ Voir le `docker-compose.yml`](./DOCKER_STACK/docker-compose.yml)

---

## 6. VPN WireGuard site-à-site

J'ai choisi WireGuard plutôt qu'IPsec ou OpenVPN pour plusieurs raisons :

- **Codebase réduite** (~4 000 lignes vs des centaines de milliers pour IPsec) — moins de surface d'attaque, plus facile à auditer
- **Cryptographie moderne** : ChaCha20-Poly1305 + Curve25519 — pas de négociation d'algorithmes comme avec IPsec
- **Furtivité** : WireGuard ne répond pas aux paquets non authentifiés, le port UDP 51820 apparaît fermé à un scan externe
- **Performance** : kernel space, latence bien plus faible qu'OpenVPN en userspace

Le routage est statique : une route `10.20.10.0/24` via `10.10.20.2` côté Siège, l'inverse côté Agence. Pas de protocole de routage dynamique — suffisant pour deux sites et ça évite les injections de routes.

---

## 7. Règles de filtrage

Politique de base : **default deny** — tout ce qui n'est pas explicitement autorisé est bloqué.

| Interface | Source | Destination | Port | Action | Pourquoi |
| :--- | :--- | :--- | :--- | :---: | :--- |
| WAN | Any | WAN | UDP/51820 | ✅ Pass | Tunnel WireGuard entrant |
| LAN | LAN | DMZ | Ports_Management | ✅ Pass | Accès aux outils depuis le LAN admin |
| LAN | LAN | pfSense | 53 | ✅ Pass | DNS interne uniquement |
| LAN | LAN | !RFC1918 | Any | ✅ Pass | Internet — mais pas les autres réseaux privés |
| DMZ | DMZ | pfSense | Any | ❌ Block | La DMZ n'accède pas à l'interface de gestion |
| DMZ | DMZ | LAN | Any | ❌ Block | Règle principale : DMZ ne peut pas initier vers le LAN |
| DMZ | DMZ | Internet | Any | ✅ Pass | Mises à jour, images Docker |
| WireGuard | serveur_librenms | LAN Agence | UDP/161 | ✅ Pass | Pull SNMP depuis LibreNMS uniquement |
| WireGuard | Any | Any | Any | ❌ Block | Tout le reste bloqué |

> Les règles BLOCK sur l'interface DMZ doivent être placées **avant** les règles PASS larges — pfSense applique le premier match de haut en bas.

[→ Matrice de flux complète dans `docs/FIREWALL_RULES.md`](./docs/FIREWALL_RULES.md)

---

## 8. Screenshots & preuves de fonctionnement

Tous les screenshots sont dans `docs/images/`. `Host_A/` = site Siège, `Host_B/` = site Agence.

**Isolation réseau**
- [`proxmox_bridges.png`](./docs/images/Host_A/proxmox_bridges.png) — bridges Proxmox sur interfaces VMware séparées
- [`vmware_vnets_isolation.png`](./docs/images/Host_A/vmware_vnets_isolation.png) — VMnets Host-only par zone
- [`lxc_dmz_ip.png`](./docs/images/Host_A/lxc_dmz_ip.png) — LXC en `10.50.10.10` sur `vmbr1`

**Filtrage**
- [`pfsense_zero_trust_block.png`](./docs/images/Host_A/pfsense_zero_trust_block.png) — règles DMZ avec BLOCK DMZ → LAN
- [`pfsense_nat_dmz.png`](./docs/images/Host_A/pfsense_nat_dmz.png) — NAT Outbound Hybrid
- [`pfsense_aliases_ports.png`](./docs/images/Host_A/pfsense_aliases_ports.png) — aliases ports
- [`pfsense_aliases_ip.png`](./docs/images/Host_B/pfsense_aliases_ip.png) — aliases IP côté Agence

**VPN & connectivité**
- [`wireguard_status_up.png`](./docs/images/Host_B/wireguard_status_up.png) — tunnel up, handshake actif
- [`pfsense_static_route_vpn.png`](./docs/images/Host_A/pfsense_static_route_vpn.png) — route statique vers l'Agence
- [`validation_traceroute_vpn.png`](./docs/images/Host_A/validation_traceroute_vpn.png) — traceroute Siège → Agence (2 sauts)
- [`validation_traceroute_dmz.png`](./docs/images/Host_A/validation_traceroute_dmz.png) — traceroute LAN → DMZ avec résolution DNS
- [`app_access_ok.png`](./docs/images/Host_B/app_access_ok.png) — accès à LibreNMS depuis le client Agence

**Supervision**
- [`dns_host_overrides.png`](./docs/images/Host_A/dns_host_overrides.png) — host overrides Unbound
- [`docker_stack_status.png`](./docs/images/Host_A/docker_stack_status.png) — tous les conteneurs up
- [`snmp_binding_secure.png`](./docs/images/Host_B/snmp_binding_secure.png) — SNMP bindé sur LAN et VPN uniquement
- [`pfsense_rules_vpn_in.png`](./docs/images/Host_B/pfsense_rules_vpn_in.png) — règle SNMP restreinte à `serveur_librenms`

---

## 9. Ce qui reste à faire

**Court terme**
- Désactiver l'auth SSH par mot de passe sur pfSense (clés Ed25519 déployées via Ansible)
- Migrer SNMP vers la v3 (auth SHA + chiffrement AES)
- Corriger l'ordre des règles sur l'interface DMZ (remonter les BLOCK)
- Supprimer la "Default allow LAN to any rule" résiduelle
- Activer Grafana et connecter LibreNMS

**Moyen terme**
- Intégrer ntopng dans Grafana
- Remplir NetBox à 100%
- Configurer les alertes LibreNMS (VPN down, espace disque)
- Push automatique des configs Oxidized vers GitHub

**Plus ambitieux**
- Proxmox SDN avec VXLAN
- NAC 802.1X avec RADIUS
- Cluster pfSense actif/passif (CARP + pfsync)

---

## 10. Ce que ce projet m'a appris

Au-delà de la technique, ce qui m'a le plus apporté c'est d'avoir résolu de vrais problèmes — pas des exercices sur papier.

La gestion des tags VLAN en environnement nested m'a forcé à comprendre pourquoi le trunk peut être dangereux dans ce contexte. Débugger un SNMP qui ne remontait pas m'a appris à lire les règles firewall dans le bon sens. Mettre en place WireGuard et valider avec des traceroutes m'a donné une vraie compréhension du routage inter-sites.

**Réseau & Sécurité**
- Segmenter un réseau avec des zones de confiance différentes et les faire cohabiter correctement
- Configurer pfSense avec des règles précises et une politique default deny réelle
- Déployer WireGuard site-à-site et comprendre pourquoi le routage statique est suffisant ici
- Identifier et mitiger les risques liés à la virtualisation imbriquée

**Supervision & Ops**
- Mettre en place LibreNMS avec supervision SNMP, graphes et alerting
- Utiliser NetBox comme inventaire réseau centralisé
- Versionner des configurations réseau avec Oxidized
- Orchestrer des services avec Docker Compose dans un LXC contraint

**Automatisation**
- Écrire des playbooks Ansible pour auditer et durcir des équipements réseau
- Déployer des clés SSH via Ansible
- Mettre en place un pipeline CI/CD pour valider les fichiers de config (GitHub Actions)
