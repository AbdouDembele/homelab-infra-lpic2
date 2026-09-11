# 🖧 Homelab Infra — Proxmox, PFsense, AdGuard Home & Passbolt

Projet de laboratoire réseau réalisé dans le cadre de ma préparation LPIC-2, visant à construire une infrastructure virtualisée segmentée : hyperviseur, pare-feu, filtrage DNS et gestionnaire de mots de passe d'équipe.

## 🎯 Objectifs du projet

- Mettre en place un hyperviseur **Proxmox VE** avec plusieurs réseaux isolés (bridges dédiés)
- Déployer un pare-feu **PFsense** en coupure entre le réseau WAN et un LAN virtuel privé
- Héberger des services internes (**AdGuard Home**, **Passbolt**) uniquement accessibles depuis le LAN protégé
- Configurer le **NAT / port forwarding** pour exposer ces services de manière contrôlée
- Valider l'ensemble par des tests de connectivité (ping, accès HTTP)

## 🧱 Stack technique

| Composant | Rôle | Technologie |
|---|---|---|
| Hyperviseur | Virtualisation | Proxmox VE 8.4 |
| Pare-feu / Routeur | Segmentation réseau, NAT | PFsense CE 2.8.0 |
| Filtrage DNS | Blocage pub/tracking, DNS local | AdGuard Home (LXC Debian 12) |
| Gestionnaire de secrets | Coffre-fort de mots de passe d'équipe | Passbolt CE (LXC Debian 12) |

## 🗺️ Architecture réseau

```
                 Internet
                     │
              ┌──────┴──────┐
              │   PC Hôte    │  192.168.1.100
              └──────┬──────┘
                     │
        vmbr0 (192.168.1.0/24 — réseau physique)
                     │
        ┌────────────┴────────────┐
        │        Proxmox           │  WAN: 192.168.1.2
        │                           │  LAN: 10.0.1.100
        └────────────┬─────────────┘
                     │
       ┌─────────────┴──────────────┐
       │     VM PFsense (pare-feu)    │
       │  WAN : 192.168.1.3            │
       │  LAN : 10.0.1.10               │
       └─────────────┬──────────────┘
                     │
        vmbr2 (10.0.1.0/16 — LAN privé)
           ┌─────────┴─────────┐
           │                   │
   ┌───────┴───────┐   ┌───────┴───────┐
   │  AdGuard Home  │   │   Passbolt     │
   │  10.0.1.1       │   │   10.0.1.2      │
   └───────────────┘   └───────────────┘
```

Le LAN `10.0.1.0/16` n'est **pas routable directement** depuis le PC hôte : tout accès aux services internes passe obligatoirement par PFsense via des règles de **NAT / port forwarding**.

## ⚙️ Étapes de mise en œuvre

### 1. Installation et configuration de Proxmox
- Installation de Proxmox VE avec IP `192.168.1.1`
- Création d'un second bridge réseau `vmbr2` dédié au LAN privé (`10.0.1.0/16`) dans `/etc/network/interfaces`

### 2. Déploiement de PFsense
- Création d'une VM avec l'ISO PFsense, deux cartes réseau (WAN sur `vmbr0`, LAN sur `vmbr2`)
  <img width="1522" height="595" alt="image" src="https://github.com/user-attachments/assets/761db3c8-d490-4e02-86fb-6ca34a6ddfde" />

- Installation CE, assignation des interfaces WAN/LAN
- Création d'une règle firewall (`easyrule`) pour débloquer l'accès à l'interface d'administration WAN

### 3. Déploiement d'AdGuard Home
- Conteneur LXC Debian 12, avec accès temporaire au WAN (pour l'installation) puis restreint au LAN
- Installation via le script officiel AdGuard Home
- Configuration de l'interface web et du serveur DNS sur `10.0.1.1`
- Mise en place d'une règle **NAT port forwarding** sur PFsense (`WAN:4444` → `10.0.1.1:80`) pour l'administration à distance

### 4. Déploiement de Passbolt
- Conteneur LXC Debian 12 dédié, IP `10.0.1.2`
- Installation via le script officiel Passbolt (base MariaDB, clé serveur)
- Règle NAT équivalente (`WAN:5555` → `10.0.1.2:80`)

## 🐛 Problème rencontré & résolution

Après configuration du port forwarding, l'accès initial à Passbolt échouait : le navigateur tentait de charger les ressources statiques (JS, favicon) directement depuis `http://10.0.1.2/...`, une IP non joignable depuis le PC hôte car située derrière le NAT de PFsense.

**Cause** : le paramètre `fullBaseUrl` dans `/etc/passbolt/passbolt.php` pointait vers l'IP interne du conteneur au lieu de l'adresse publique utilisée pour y accéder.

**Solution** : modification de `fullBaseUrl` pour utiliser `http://192.168.1.3:5555` (IP + port exposés côté PFsense), ce qui a permis à toutes les ressources de se charger correctement.

## ✅ Tests de validation

**Connectivité interne (depuis Proxmox) :**
```
ping 10.0.1.10   → OK (PFsense LAN)
ping 10.0.1.1    → OK (AdGuard)
ping 10.0.1.2    → OK (Passbolt)
```

**Accès aux services depuis le PC hôte (via NAT PFsense) :**
- `http://192.168.1.3:4444` → interface AdGuard Home
- `http://192.168.1.3:5555` → interface Passbolt

## 📋 Tableau récapitulatif des adresses IP

| Machine | Interface WAN | Interface LAN |
|---|---|---|
| PC Hôte | 192.168.1.100 | — |
| Proxmox | 192.168.1.2 | 10.0.1.100 |
| PFsense | 192.168.1.3 | 10.0.1.10 |
| AdGuard Home | — | 10.0.1.1 |
| Passbolt | — | 10.0.1.2 |

## 📸 Captures d'écran

Les captures détaillées de chaque étape (installation, configuration, tests) sont disponibles dans le dossier [`docs/screenshots/`](./docs/screenshots).

## 🔧 Compétences mises en pratique

- Virtualisation et administration Proxmox VE (VM, LXC, bridges réseau)
- Segmentation réseau et sécurité périmétrique avec un pare-feu (PFsense)
- NAT / Port forwarding et règles de filtrage
- Déploiement et durcissement de services applicatifs (AdGuard Home, Passbolt)
- Diagnostic et résolution de problèmes réseau (NAT reflection, résolution d'URL applicative)

---

*Projet réalisé dans le cadre d'une préparation à la certification LPIC-2.*
