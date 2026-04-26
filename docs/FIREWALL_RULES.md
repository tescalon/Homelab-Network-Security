# Politique de Sécurité Pare-feu — Matrice de Flux Détaillée

Ce document détaille l'ensemble des règles de filtrage configurées sur le pfSense Siège. La politique globale est **Default Deny** (Zero Trust) : tout trafic non explicitement autorisé est bloqué.

---

## Aliases utilisés dans les règles

### Aliases IP

| Alias | Type | Valeur(s) | Usage |
| :--- | :--- | :--- | :--- |
| `DMZ` | Network | `10.50.10.0/24` | Zone DMZ SecOps |
| `Siege` | Network | `10.10.10.0/24` | LAN Siège |
| `RFC1918` | Network | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Toutes les plages privées |
| `serveur_librenms` | Host | `10.50.10.10` | Serveur de supervision LibreNMS |

### Aliases Ports

| Alias | Ports | Usage |
| :--- | :--- | :--- |
| `Ports_Management` | `80, 443, 22, 8000, 3000, 8888` | Accès aux outils GRC (HTTP, HTTPS, SSH, NetBox, Grafana, Oxidized) |
| `Ports_Infra` | `53, 123, 161` | DNS, NTP, SNMP |

---

## Interface WAN

**Politique :** Deny All entrant par défaut (règle pfSense implicite). Seul le port WireGuard est ouvert.

| # | État | Protocole | Source | Destination | Port | Action | Justification |
| :--- | :--- | :--- | :--- | :--- | :--- | :---: | :--- |
| 1 | ✅ | UDP | Any | WAN Address | 51820 | **PASS** | Établissement du tunnel WireGuard Site-à-Site. Seul port entrant autorisé. |
| — | — | * | Any | Any | * | **BLOCK** | Règle implicite pfSense — tout le reste est rejeté. |

---

## Interface LAN Siège

**Politique :** Zone de confiance (admin). Les flux vers Internet sont autorisés, les retours vers les réseaux privés sont filtrés.

| # | État | Protocole | Source | Destination | Port | Action | Justification |
| :--- | :--- | :--- | :--- | :--- | :--- | :---: | :--- |
| 1 | ✅ | * | * | LAN Address | 80 | **PASS** | Anti-Lockout Rule — Empêche de se couper l'accès à l'interface pfSense. |
| 2 | ✅ | TCP | LAN subnets | DMZ | `Ports_Management` | **PASS** | Accès administrateur aux outils GRC depuis le LAN. |
| 3 | ✅ | TCP/UDP | LAN subnets | LAN Address | 53 | **PASS** | Résolution DNS locale via Unbound (pfSense). |
| 4 | ✅ | * | LAN subnets | `!RFC1918` | * | **PASS** | Accès Internet uniquement. Le `!` exclut toutes les plages privées, empêchant le rebond vers d'autres zones. |
| — | — | * | Any | Any | * | **BLOCK** | Default Deny implicite. |

> **Note :** La règle "Default allow LAN to any rule" générée par pfSense à l'installation doit être **supprimée** pour que la règle `!RFC1918` soit effective. Sans cette suppression, tout le trafic LAN passerait avant d'atteindre les règles de restriction.

---

## Interface SECOPS_DMZ

**Politique :** Zone semi-ouverte. La DMZ peut accéder à Internet mais **ne peut jamais initier de connexion vers le LAN admin**. C'est le cœur du modèle Zero Trust.

| # | État | Protocole | Source | Destination | Port | Action | Justification |
| :--- | :--- | :--- | :--- | :--- | :--- | :---: | :--- |
| 1 | ✅ | TCP | LAN subnets | SECOPS_DMZ subnets | `Ports_Management` | **PASS** | Permet l'accès entrant depuis le LAN vers les services DMZ. |
| 2 | ❌ | * | Any | This Firewall (self) | * | **BLOCK** | **Isolation critique.** La DMZ ne peut pas accéder directement à l'interface de management pfSense. |
| 3 | ❌ | * | SECOPS_DMZ subnets | LAN subnets | * | **BLOCK** | **Règle Zero Trust fondamentale.** Un service compromis en DMZ ne peut pas atteindre le LAN admin. |
| 4 | ✅ | TCP/UDP | SECOPS_DMZ subnets | Any | * | **PASS** | Accès sortant Internet (mises à jour images Docker, repositories). |
| 5 | ✅ | ICMP | SECOPS_DMZ subnets | Any | — | **PASS** | Diagnostic réseau (ping depuis la DMZ). |
| 6 | ❌ | * | Any | Any | * | **BLOCK** | Default Deny explicite. |

> **Ordre critique :** Les règles BLOCK (lignes 2 et 3) doivent impérativement être **au-dessus** de toute règle PASS large. pfSense applique les règles du haut vers le bas, premier match gagne.

---

## Interface VPN (WireGuard)

**Politique :** L'Agence peut accéder aux outils GRC en lecture/supervision. Le Siège peut superviser l'Agence via SNMP. Tout le reste est bloqué.

| # | État | Protocole | Source | Destination | Port | Action | Justification |
| :--- | :--- | :--- | :--- | :--- | :--- | :---: | :--- |
| 1 | ✅ | UDP | `serveur_librenms` | LAN Address | 161 (SNMP) | **PASS** | Pull SNMP de LibreNMS vers pfSense Agence via VPN. Source restreinte à l'alias `serveur_librenms` (10.50.10.10 uniquement — principe moindre privilège). |
| 2 | ✅ | ICMP | `Siege` | Any | — | **PASS** | Ping de supervision depuis le réseau Siège. |
| 3 | ❌ | * | Any | Any | * | **BLOCK** | Default Deny — tout trafic VPN non autorisé est rejeté. |

---

## Résumé — Matrice de flux inter-zones

| Source \ Destination | WAN | LAN Siège | DMZ SecOps | VPN/Agence | Internet |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **WAN** | — | ❌ | ❌ | ❌ (UDP/51820 ✅) | — |
| **LAN Siège** | ❌ | ✅ | ✅ (Ports_Mgmt) | ✅ (ICMP) | ✅ |
| **DMZ SecOps** | ❌ | ❌ | ✅ | ❌ | ✅ |
| **VPN/Agence** | ❌ | ❌ | ❌ | ✅ | ❌ |
| **Agence → DMZ** | ❌ | ❌ | ✅ (via route VPN) | — | ❌ |
