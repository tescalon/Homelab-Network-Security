# Architecture Technique Détaillée

Ce document décrit en détail l'architecture de l'infrastructure, les choix techniques retenus, et les configurations déployées. Il sert de référence technique (DAT) pour le projet.

---

## 1. Environnement d'Hébergement

### Infrastructure physique réelle

Le projet tourne sur **deux machines physiques distinctes**, simulant un vrai déploiement multi-site.

| Machine | Rôle | Hyperviseur | Réseau |
| :--- | :--- | :--- | :--- |
| **Machine Principale** | Site Siège (HQ) | VMware Workstation + Proxmox VE (nested) | VMnet1 (LAN HQ), VMnet2 (DMZ), VMnet8 (NAT/WAN) |
| **Machine Secondaire** | Site Agence (Branch) | VMware Workstation | VMnet1 (LAN Agence), VMnet8 (NAT/WAN) |

> **Note sur la virtualisation imbriquée :** Proxmox VE tourne lui-même comme VM dans VMware Workstation sur la machine principale. L'isolation L2 réelle entre les zones (LAN, DMZ) est assurée par les VMnets VMware en mode "Host-only" (isolés du réseau physique). Les Linux Bridges Proxmox (`vmbr0`, `vmbr1`) s'appuient ensuite sur ces interfaces VMware (`ens33`, `ens37`).

### Plan de virtualisation (Machine Principale / HQ)

```
[Machine Physique HQ]
└── VMware Workstation
    ├── VMnet1 (Host-only) → 10.10.10.0/24   [LAN Siège]
    ├── VMnet2 (Host-only) → 10.50.10.0/24   [DMZ SecOps]
    └── VMnet8 (NAT)       → 192.168.x.x     [WAN simulé]
        │
        ├── VM : Proxmox VE (nested hypervisor)
        │   ├── vmbr0 → ens33 (VMnet1) → LAN Siège
        │   ├── vmbr1 → ens37 (VMnet2) → DMZ
        │   │
        │   └── LXC 105 : srv-admin-hq
        │       ├── eth0 → vmbr1 → 10.50.10.10/24
        │       └── Services Docker :
        │           ├── NetBox    :8000
        │           ├── LibreNMS  :80
        │           ├── Oxidized  :8888
        │           └── Cloudflared (tunnel ZeroTrust)
        │
        └── VM : pfSense Siège
            ├── em0 → WAN (DHCP/NAT VMware)
            ├── em1 → vmbr0 → 10.10.10.254/24 (LAN)
            ├── em2 → vmbr1 → 10.50.10.254/24 (SECOPS/DMZ)
            └── em3 → tun_wg0 → 10.10.20.1/24 (WireGuard)
```

---

## 2. Plan d'Adressage IP (IPAM)

L'adressage respecte la RFC1918 avec une logique géographique : `10.10.x.x` pour le Siège, `10.20.x.x` pour l'Agence, `10.50.x.x` pour la DMZ.

| Zone | Réseau CIDR | Gateway | Éléments clés |
| :--- | :--- | :--- | :--- |
| **LAN Siège** | `10.10.10.0/24` | `10.10.10.254` (pfSense em1) | Proxmox : `10.10.10.15` |
| **DMZ SecOps** | `10.50.10.0/24` | `10.50.10.254` (pfSense em2) | srv-admin-hq : `10.50.10.10` |
| **LAN Agence** | `10.20.10.0/24` | `10.20.10.254` (pfSense Agence em1) | Client Debian : `10.20.10.130` |
| **Tunnel VPN** | `10.10.20.0/24` | — | Peer HQ : `10.10.20.1` / Peer Agence : `10.10.20.2` |

---

## 3. Interfaces pfSense Siège

| Interface | Nom interne | IP / CIDR | Zone de sécurité | Rôle |
| :--- | :--- | :--- | :--- | :--- |
| `em0` | WAN | DHCP (NAT VMware) | *Untrusted* | Accès Internet simulé |
| `em1` | LAN | `10.10.10.254/24` | *Trusted* | Zone admin / gestion |
| `em2` | SECOPS_DMZ | `10.50.10.254/24` | *DMZ* | Zone services (Docker, monitoring) |
| `em3` / `tun_wg0` | VPN | `10.10.20.1/24` | *Overlay* | Tunnel WireGuard site-à-site |

---

## 4. pfSense Agence

Le pfSense Agence tourne sur la machine secondaire dans VMware Workstation.

| Interface | IP / CIDR | Rôle |
| :--- | :--- | :--- |
| `em0` (WAN) | `192.168.1.50/24` (DHCP) | Accès Internet / endpoint WireGuard |
| `em1` (LAN) | `10.20.10.254/24` | LAN Agence |
| `tun_wg0` (VPN) | `10.10.20.2/24` | Peer WireGuard (côté Agence) |

**Package installé :** `ntopng` — analyseur de flux réseau déployé directement sur le pfSense Agence (package natif FreeBSD). Ce positionnement "Edge" lui permet d'analyser **tout le trafic** entrant et sortant de l'Agence avant routage, offrant une visibilité maximale sur les flux.

---

## 5. Interconnexion WireGuard (Site-à-Site)

### Choix technologique vs alternatives

| Critère | WireGuard | IPsec | OpenVPN |
| :--- | :--- | :--- | :--- |
| Lignes de code | ~4 000 | ~400 000 | ~100 000 |
| Surface d'attaque | Minimale | Importante | Moyenne |
| Performance | Très élevée (kernel) | Élevée | Moyenne |
| Furtivité | Oui (pas de réponse non-auth) | Non | Non |
| Cryptographie | ChaCha20 / Curve25519 | Variable | Variable |

### Configuration retenue

- **Routage :** Statique (route `10.20.10.0/24` via `vpn-10.10.20.2` sur pfSense Siège)
- **Port :** UDP/51820
- **MTU :** 1500 (valeur par défaut, suffisante en environnement virtualisé)
- **Handshake interval :** Standard WireGuard (~25s keepalive)

---

## 6. Optimisations Kernel pfSense

Configurations appliquées dans `System > Advanced > Networking` :

| Option | État | Justification |
| :--- | :--- | :--- |
| Hardware Checksum Offload | ✅ Activé | Non nécessaire à désactiver avec VirtIO récent |
| Hardware TCP Segmentation Offload (TSO) | ❌ Désactivé | Incompatibilité avec certains drivers paravirtualisés |
| Hardware Large Receive Offload (LRO) | ❌ Désactivé | Peut causer des corruptions de paquets en environnement virtualisé |
| hn ALTQ support | ✅ Activé | Requis pour le traffic shaping sur interfaces VirtIO de type `hn` |

---

## 7. Stack Docker (srv-admin-hq : 10.50.10.10)

Tous les services GRC tournent dans des conteneurs Docker orchestrés par `docker-compose.yml`, hébergés dans un LXC non-privilégié sur Proxmox (isolation kernel entre l'hyperviseur et les services).

| Service | Image | Port exposé | Rôle |
| :--- | :--- | :--- | :--- |
| `netbox` | `netboxcommunity/netbox:latest` | `8000` | IPAM / CMDB (Source of Truth) |
| `netbox-worker` | `netboxcommunity/netbox:latest` | — | Worker tâches asynchrones NetBox |
| `postgres` | `postgres:15-alpine` | — | BDD NetBox (interne) |
| `redis` | `redis:7-alpine` | — | Cache NetBox (interne) |
| `librenms` | `librenms/librenms:latest` | `80` | Supervision SNMP / Alerting |
| `mariadb` | `mariadb:10.11` | — | BDD LibreNMS (interne) |
| `oxidized` | `oxidized/oxidized:latest` | `8888` | Backup automatique configs routeurs |
| `cloudflared` | `cloudflare/cloudflared:latest` | — | Tunnel Zero Trust (accès distant sans port public) |

### Réseau Docker

Tous les services partagent un réseau bridge interne `homelab-net`. Seuls les ports explicitement définis dans `ports:` sont exposés sur l'interface hôte (`10.50.10.10`). Les bases de données (postgres, mariadb, redis) ne sont jamais exposées directement.

---

## 8. DNS Interne (pfSense Unbound)

Le resolver DNS Unbound tourne en mode récursif sur pfSense Siège avec les host overrides suivants :

| Hostname | Domaine | IP résolue | Service |
| :--- | :--- | :--- | :--- |
| `netbox` | `homelab.lan` | `10.50.10.10` | NetBox (Docker) |
| `librenms` | `homelab.lan` | `10.50.10.10` | LibreNMS (Docker) |

Avantage : les clients du LAN utilisent `librenms.homelab.lan` au lieu de `10.50.10.10:80`, rendant l'infrastructure indépendante des IPs en cas de reconfiguration.

---

## 9. Supervision SNMP

> **Note technique :** La supervision est actuellement configurée en **SNMPv2c** avec une community string non-standard (`aqwzsxedcrfv`). Le flux SNMP transite exclusivement dans le tunnel WireGuard chiffré, ce qui compense l'absence de chiffrement natif SNMPv2c. La migration vers **SNMPv3** (auth SHA + chiffrement AES) est planifiée en roadmap.

| Direction du flux | Source | Destination | Port | Protocole |
| :--- | :--- | :--- | :--- | :--- |
| Pull SNMP HQ → Agence | `10.50.10.10` (LibreNMS) | `10.20.10.254` (pfSense Agence) | UDP/161 | SNMPv2c via WireGuard |

La règle WireGuard sur pfSense Siège autorise uniquement `serveur_librenms` (alias IP `10.50.10.10`) à interroger `LAN address:161`, principe du moindre privilège.
