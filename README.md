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
- <img width="1459" height="593" alt="image" src="https://github.com/user-attachments/assets/80783f5b-ec47-4ac3-8af4-2bfd4f8ae472" />
<img width="1459" height="701" alt="image" src="https://github.com/user-attachments/assets/11d60748-8b10-4799-a3a5-6a924514101c" />

- Création d'une règle firewall (`easyrule`) pour débloquer l'accès à l'interface d'administration WAN
<img width="1459" height="829" alt="image" src="https://github.com/user-attachments/assets/f31cdb4b-24ea-4dac-b161-e1fda5fe7e45" />

<img width="1459" height="574" alt="image" src="https://github.com/user-attachments/assets/cacd2370-8caa-4499-83fe-3650623a4136" />

### 3. Déploiement d'AdGuard Home
- Conteneur LXC Debian 12, avec accès temporaire au WAN (pour l'installation) puis restreint au LAN
- <img width="1236" height="916" alt="image" src="https://github.com/user-attachments/assets/0220c85c-7501-437e-9d05-67964fe82233" />
<img width="1127" height="859" alt="image" src="https://github.com/user-attachments/assets/bce71951-ec4e-4d54-8449-9c7a0d7d6491" />
  
- Installation via le script officiel AdGuard Home
- Configuration de l'interface web et du serveur DNS sur `10.0.1.1`
  <img width="1008" height="945" alt="image" src="https://github.com/user-attachments/assets/8bce5650-84c2-4d17-b861-74d105037806" />

- Mise en place d'une règle **NAT port forwarding** sur PFsense (`WAN:4444` → `10.0.1.1:80`) pour l'administration à distance
<img width="1063" height="292" alt="image" src="https://github.com/user-attachments/assets/74ace073-6982-4fb4-a180-3553188be910" />
<img width="1459" height="778" alt="image" src="https://github.com/user-attachments/assets/c2cab920-5374-44dc-bc1e-30191a621375" />
<img width="1459" height="876" alt="image" src="https://github.com/user-attachments/assets/bb9ec3b4-b677-47da-a1f8-0058b4068596" />

<img width="1459" height="568" alt="image" src="https://github.com/user-attachments/assets/f07daacb-8ab3-44d2-8c91-ff9f2e6825a7" />




### 4. Déploiement de Passbolt
- Conteneur LXC Debian 12 dédié, IP `10.0.1.2`
<img width="1459" height="876" alt="image" src="https://github.com/user-attachments/assets/4b72a1c3-4fc9-4839-b3be-d41e1b81f76e" />

- Installation via le script officiel Passbolt (base MariaDB, clé serveur)
  <img width="1459" height="549" alt="image" src="https://github.com/user-attachments/assets/dffb34ab-4d6b-43d0-8dbe-346c0afbf232" />
<img width="1459" height="727" alt="image" src="https://github.com/user-attachments/assets/7db18260-d2e3-4fa9-a36a-207bfb03b793" />

- Règle NAT équivalente (`WAN:5555` → `10.0.1.2:80`)
  <img width="1226" height="945" alt="image" src="https://github.com/user-attachments/assets/e8c9a58b-9c7e-4534-93c1-c95554195f60" />


## 🐛 Problème rencontré & résolution

Après configuration du port forwarding, l'accès initial à Passbolt échouait : le navigateur tentait de charger les ressources statiques (JS, favicon) directement depuis `http://10.0.1.2/...`, une IP non joignable depuis le PC hôte car située derrière le NAT de PFsense.

<img width="1459" height="356" alt="image" src="https://github.com/user-attachments/assets/b942cbf2-32be-4c4f-9f52-c01e46934208" />


**Cause** : le paramètre `fullBaseUrl` dans `/etc/passbolt/passbolt.php` pointait vers l'IP interne du conteneur au lieu de l'adresse publique utilisée pour y accéder.

**Solution** : modification de `fullBaseUrl` pour utiliser `http://192.168.1.3:5555` (IP + port exposés côté PFsense), ce qui a permis à toutes les ressources de se charger correctement.

<img width="1459" height="586" alt="image" src="https://github.com/user-attachments/assets/339a9bd4-a557-49f2-a36b-1afc0ef71708" />


<img width="1459" height="571" alt="image" src="https://github.com/user-attachments/assets/251bd38a-4e5e-4047-9aca-49119e603d85" />


## ✅ Tests de validation

**Connectivité interne (depuis Proxmox) :**
```
ping 10.0.1.10   → OK (PFsense LAN)
ping 10.0.1.1    → OK (AdGuard)
ping 10.0.1.2    → OK (Passbolt)
```
<img width="892" height="945" alt="image" src="https://github.com/user-attachments/assets/f889fcd2-5359-4c9a-8327-67fa5306b0b1" />


**Accès aux services depuis le PC hôte (via NAT PFsense) :**
- `http://192.168.1.3:4444` → interface AdGuard Home
  <img width="884" height="945" alt="image" src="https://github.com/user-attachments/assets/658fcf89-dd8b-4635-ab0e-9c9bfc943780" />

- `http://192.168.1.3:5555` → interface Passbolt
<img width="888" height="945" alt="image" src="https://github.com/user-attachments/assets/8b788382-d88c-497c-82c0-ea74b0af54fd" />


## 📋 Tableau récapitulatif des adresses IP

| Machine | Interface WAN | Interface LAN |
|---|---|---|
| PC Hôte | 192.168.1.100 | — |
| Proxmox | 192.168.1.2 | 10.0.1.100 |
| PFsense | 192.168.1.3 | 10.0.1.10 |
| AdGuard Home | — | 10.0.1.1 |
| Passbolt | — | 10.0.1.2 |


## 🔧 Compétences mises en pratique

- Virtualisation et administration Proxmox VE (VM, LXC, bridges réseau)
- Segmentation réseau et sécurité périmétrique avec un pare-feu (PFsense)
- NAT / Port forwarding et règles de filtrage
- Déploiement et durcissement de services applicatifs (AdGuard Home, Passbolt)
- Diagnostic et résolution de problèmes réseau (NAT reflection, résolution d'URL applicative)

---

*Projet réalisé dans le cadre d'une préparation à la certification LPIC-2.*
