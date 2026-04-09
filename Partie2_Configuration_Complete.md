# PARTIE 2 : CONFIGURATION COMPLÈTE

## Table des Matières
1. [Analyse de la Topologie](#analyse-de-la-topologie)
2. [Plan d'Adressage Corrigé](#plan-dadressage-corrigé)
3. [Configuration du Siège](#configuration-du-siège)
4. [Configuration des Branches](#configuration-des-branches)
5. [Configuration du Backbone MPLS](#configuration-du-backbone-mpls)
6. [Vérification et Tests](#vérification-et-tests)

---

## Analyse de la Topologie

### Architecture Réelle de la Topologie Partie 2

```
                    ┌─────── SIÈGE ───────┐
                    │                     │
    ┌───────────────┴─────────────────────┴──────────────┐
    │                                                     │
    │         Fédérateur1 ←→ Fédérateur2               │
    │      (e0/0, e0/1, e0/2, e0/3, e1/0)              │
    │            ↓            ↓     ↓     ↓             │
    │           g0/1         g0/2  g0/1  g0/2           │
    │            ↓            ↓     ↓     ↓             │
    │           SW1          SW2   SW1   SW2            │
    │           ↓            ↓                           │
    │          PC0          PC1                          │
    │                                                     │
    │     CE11 (Siège1) - CE21 (Siège2)                │
    │     (f0/0)              (f0/0)                     │
    │        ↓                   ↓                       │
    └────────┼───────────────────┼─────────────────────┘
             │                   │
             └───────────┬───────┘
                         ↓
        ┌──────────────────────────────────┐
        │  BACKBONE MPLS (Area 0)          │
        │  PE1 ←→ P1 ←→ P2 ←→ PE2        │
        │ OSPF + MPLS + MP-BGP             │
        └──────────────────────────────────┘
             ↓           ↓
        (g3/0, g4/0)  (g3/0, g4/0)
             ↓           ↓
        CE11, CE21   CE12, CE22
             │           │
        ┌────┘           └────┐
        │                     │
        │                     │
    ┌───┴──────┐         ┌────┴───┐
    │   SW3    │         │   SW4  │
    │  CE12    │         │  CE22  │
    │  (g0/0)  │         │(g0/0)  │
    │  (g0/1)  │         │(g0/1)  │
    │   ↓      │         │   ↓    │
    │  PC2     │         │  PC3   │
    └──────────┘         └────────┘
```

### Connexions Détaillées

| Équipement 1 | Interface 1 | Équipement 2 | Interface 2 | Type |
|-------------|-----------|-------------|-----------|------|
| Fédérateur1 | e0/0, e0/1, e0/2, e0/3 | Fédérateur2 | e0/0, e0/1, e0/2, e0/3 | Trunks multiples |
| Fédérateur1 | e1/0 | Fédérateur2 | e1/0 | EtherChannel |
| Fédérateur1 | g0/1 | SW1 | g0/1 | Trunk |
| Fédérateur1 | g0/2 | SW2 | g0/2 | Trunk |
| Fédérateur2 | g0/1 | SW1 | g0/2 | Trunk |
| Fédérateur2 | g0/2 | SW2 | g0/1 | Trunk |
| SW1 | g0/0 | PC0 | e0 | Access |
| SW2 | g0/0 | PC1 | e0 | Access |
| Fédérateur1 | f0/0 | CE11 | f0/0 | WAN |
| Fédérateur1 | f0/0 | CE21 | f0/0 | WAN |
| CE11 | g1/0 | - | - | Loopback |
| CE21 | g1/0 | - | - | Loopback |
| PE1 | g1/0 | P1 | g2/0 | Backbone |
| PE1 | g2/0 | P2 | g3/0 | Backbone |
| PE2 | g1/0 | P1 | g3/0 | Backbone |
| PE2 | g2/0 | P2 | g2/0 | Backbone |
| P1 | g1/0 | P2 | g1/0 | Backbone |
| PE1 | g3/0 | CE11 | f0/0 | Client |
| PE1 | g4/0 | CE21 | f0/0 | Client |
| PE2 | g3/0 | CE12 | f0/0 | Client |
| PE2 | g4/0 | CE22 | f0/0 | Client |
| CE12 | g1/0 | SW3 | g0/1 | Trunk |
| SW3 | g0/0 | PC2 | e0 | Access |
| CE22 | g1/0 | SW4 | g0/1 | Trunk |
| SW4 | g0/0 | PC3 | e0 | Access |

---

## Plan d'Adressage Corrigé

### Backbone MPLS (Area 0)
| Équipement | Interface | Adresse | Masque | Description |
|-----------|-----------|---------|--------|-------------|
| PE1 | Loopback 0 | 1.1.1.1 | /32 | Router ID PE1 |
| PE2 | Loopback 0 | 2.2.2.2 | /32 | Router ID PE2 |
| P1 | Loopback 0 | 3.3.3.3 | /32 | Router ID P1 |
| P2 | Loopback 0 | 4.4.4.4 | /32 | Router ID P2 |
| PE1-P1 | 10.1.1.0 | /30 | - | PE1 g1/0 - P1 g2/0 |
| PE1-P2 | 10.1.1.4 | /30 | - | PE1 g2/0 - P2 g3/0 |
| PE2-P1 | 10.1.1.12 | /30 | - | PE2 g3/0 - P1 g3/0 |
| PE2-P2 | 10.1.1.8 | /30 | - | PE2 g1/0 - P2 g2/0 |
| P1-P2 | 10.1.1.20 | /30 | - | P1 g1/0 - P2 g1/0 |

### Siège - Fédérateurs
| Équipement | Interface | VLAN | Adresse | Masque | Fonction |
|-----------|-----------|------|---------|--------|----------|
| Fédérateur1 | VLAN 10 | 10 | 172.16.10.1 | /24 | DATA1 (Gateway principal) |
| Fédérateur1 | VLAN 15 | 15 | 172.16.15.1 | /24 | DATA2 (Gateway principal) |
| Fédérateur1 | VLAN 20 | 20 | 172.16.20.1 | /24 | Management (Gateway principal) |
| Fédérateur1 | VLAN 30 | 30 | 192.168.1.1 | /30 | Trunk WAN (OSPF) |
| Fédérateur2 | VLAN 10 | 10 | 172.16.10.2 | /24 | DATA1 (Gateway secondaire) |
| Fédérateur2 | VLAN 15 | 15 | 172.16.15.2 | /24 | DATA2 (Gateway secondaire) |
| Fédérateur2 | VLAN 20 | 20 | 172.16.20.2 | /24 | Management (Gateway secondaire) |
| Fédérateur2 | VLAN 30 | 30 | 192.168.1.2 | /30 | Trunk WAN (OSPF) |
| SW1 | VLAN 20 | 20 | 172.16.20.101 | /24 | Management SW1 |
| SW2 | VLAN 20 | 20 | 172.16.20.102 | /24 | Management SW2 |

### Siège - Routeurs Clients
| Équipement | Interface | Adresse | Masque | OSPF Area |
|-----------|-----------|---------|--------|-----------|
| CE11 (Siège1) | Loopback 0 | 172.16.11.11 | /32 | 11 |
| CE11 | f0/0 | 192.168.1.2 | /30 | 11 |
| CE21 (Siège2) | Loopback 0 | 172.16.21.21 | /32 | 21 |
| CE21 | f0/0 | 192.168.1.6 | /30 | 21 |

### Branches
| Équipement | Interface | VLAN | Adresse | Masque | Fonction |
|-----------|-----------|------|---------|--------|----------|
| CE12 (Branch1) | Loopback 0 | - | 172.16.12.12 | /32 | Router ID |
| CE12 | g1/0.101 | 101 | 172.16.101.1 | /24 | Management Gateway |
| CE12 | g1/0.102 | 102 | 172.16.102.1 | /24 | DATA Gateway |
| CE12 | g1/0.99 | 99 | 192.168.1.10 | /30 | WAN vers PE2 |
| SW3 | VLAN 101 | 101 | 172.16.101.100 | /24 | Management SW3 |
| CE22 (Branch2) | Loopback 0 | - | 172.16.22.22 | /32 | Router ID |
| CE22 | g1/0.103 | 103 | 172.16.103.1 | /24 | Management Gateway |
| CE22 | g1/0.104 | 104 | 172.16.104.1 | /24 | DATA Gateway |
| CE22 | g1/0.99 | 99 | 192.168.1.14 | /30 | WAN vers PE2 |
| SW4 | VLAN 103 | 103 | 172.16.103.100 | /24 | Management SW4 |

---

## Configuration du Siège

### Configuration de Fédérateur1

#### 1. Configuration Basique et VLANs
```bash
enable
configure terminal
hostname Fédérateur1
!
# VLANs
vlan 10
  name DATA1
  exit
vlan 15
  name DATA2
  exit
vlan 20
  name Management
  exit
vlan 30
  name TRUNK-VLAN
  exit
```

#### 2. Configuration des Interfaces VLAN avec HSRP
```bash
# VLAN 10 - DATA1 avec HSRP
interface vlan 10
  description "DATA1 - Siège"
  ip address 172.16.10.1 255.255.255.0
  standby 1 ip 172.16.10.254
  standby 1 priority 150
  standby 1 preempt
  standby 1 timers 3 10
  no shutdown
  exit
!
# VLAN 15 - DATA2 avec HSRP
interface vlan 15
  description "DATA2 - Siège"
  ip address 172.16.15.1 255.255.255.0
  standby 2 ip 172.16.15.254
  standby 2 priority 150
  standby 2 preempt
  standby 2 timers 3 10
  no shutdown
  exit
!
# VLAN 20 - Management avec HSRP
interface vlan 20
  description "Management - Siège"
  ip address 172.16.20.1 255.255.255.0
  standby 3 ip 172.16.20.254
  standby 3 priority 150
  standby 3 preempt
  standby 3 timers 3 10
  no shutdown
  exit
!
# VLAN 30 - Liaison WAN vers Backbone
interface vlan 30
  description "WAN vers Backbone MPLS"
  ip address 192.168.1.1 255.255.255.252
  no shutdown
  exit
```

#### 3. Configuration des Ports Accès vers Switches Layer 2
```bash
# Port vers SW1
interface g0/1
  description "Port Trunk vers SW1"
  switchport mode trunk
  switchport trunk native vlan 20
  switchport trunk allowed vlan 10,15,20,30
  no shutdown
  exit
!
# Port vers SW2
interface g0/2
  description "Port Trunk vers SW2"
  switchport mode trunk
  switchport trunk native vlan 20
  switchport trunk allowed vlan 10,15,20,30
  no shutdown
  exit
!
# EtherChannel vers Fédérateur2
interface e0/0
  description "EtherChannel vers Fédérateur2 - Lien 1"
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
interface e0/1
  description "EtherChannel vers Fédérateur2 - Lien 2"
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
interface e0/2
  description "EtherChannel vers Fédérateur2 - Lien 3"
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
interface e0/3
  description "EtherChannel vers Fédérateur2 - Lien 4"
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
interface e1/0
  description "Backup EtherChannel"
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
# Port-Channel EtherChannel
interface port-channel 1
  description "EtherChannel vers Fédérateur2"
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  no shutdown
  exit
```

#### 4. Configuration DHCP sur Fédérateur1
```bash
# DHCP pour VLAN 10
ip dhcp pool VLAN10_POOL
  network 172.16.10.0 255.255.255.0
  default-router 172.16.10.254
  dns-server 8.8.8.8 8.8.4.4
  lease 1 0
  exit
!
ip dhcp excluded-address 172.16.10.1 172.16.10.10
!
# DHCP pour VLAN 15
ip dhcp pool VLAN15_POOL
  network 172.16.15.0 255.255.255.0
  default-router 172.16.15.254
  dns-server 8.8.8.8 8.8.4.4
  lease 1 0
  exit
!
ip dhcp excluded-address 172.16.15.1 172.16.15.10
```

#### 5. Configuration VRF et OSPF
```bash
# Créer le VRF
ip vrf VPN_SSIR
  description "VPN Siège et Branches"
  rd 100:1
  route-target both 100:1
  exit
!
# Appliquer VRF à VLAN 30
interface vlan 30
  ip vrf forwarding VPN_SSIR
  ip address 192.168.1.1 255.255.255.252
  no shutdown
  exit
!
# OSPF pour le VRF
router ospf 300 vrf VPN_SSIR
  router-id 1.1.1.1
  network 172.16.10.0 0.0.0.255 area 11
  network 172.16.15.0 0.0.0.255 area 11
  network 192.168.1.0 0.0.0.3 area 11
  passive-interface default
  no passive-interface vlan 30
  interface vlan 30
    ip ospf cost 1000
  exit
  exit
```

#### 6. Configuration Sécurité
```bash
enable password [NOM_2EME_BINOME]
service password-encryption
!
line console 0
  password [NOM_2EME_BINOME]
  login
  logging synchronous
  exit
!
line vty 0 4
  password [NOM_2EME_BINOME]
  login
  logging synchronous
  exit
!
logging buffered 1000
spanning-tree mode rapid
```

### Configuration de Fédérateur2

```bash
enable
configure terminal
hostname Fédérateur2
!
# VLANs identiques à Fédérateur1
vlan 10
  name DATA1
  exit
vlan 15
  name DATA2
  exit
vlan 20
  name Management
  exit
vlan 30
  name TRUNK-VLAN
  exit
!
# Interfaces VLAN avec HSRP - Priorité 100 (secondaire)
interface vlan 10
  ip address 172.16.10.2 255.255.255.0
  standby 1 ip 172.16.10.254
  standby 1 priority 100
  standby 1 preempt
  no shutdown
  exit
!
interface vlan 15
  ip address 172.16.15.2 255.255.255.0
  standby 2 ip 172.16.15.254
  standby 2 priority 100
  standby 2 preempt
  no shutdown
  exit
!
interface vlan 20
  ip address 172.16.20.2 255.255.255.0
  standby 3 ip 172.16.20.254
  standby 3 priority 100
  standby 3 preempt
  no shutdown
  exit
!
interface vlan 30
  ip vrf forwarding VPN_SSIR
  ip address 192.168.1.2 255.255.255.252
  no shutdown
  exit
!
# Ports Trunk
interface g0/1
  switchport mode trunk
  switchport trunk native vlan 20
  switchport trunk allowed vlan 10,15,20,30
  no shutdown
  exit
!
interface g0/2
  switchport mode trunk
  switchport trunk native vlan 20
  switchport trunk allowed vlan 10,15,20,30
  no shutdown
  exit
!
# EtherChannel (identique à Fédérateur1)
interface e0/0
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
interface e0/1
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
interface e0/2
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
interface e0/3
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
interface e1/0
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
interface port-channel 1
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  no shutdown
  exit
!
# VRF et OSPF identiques
ip vrf VPN_SSIR
  rd 100:1
  route-target both 100:1
  exit
!
router ospf 300 vrf VPN_SSIR
  router-id 1.1.1.1
  network 172.16.10.0 0.0.0.255 area 11
  network 172.16.15.0 0.0.0.255 area 11
  network 192.168.1.0 0.0.0.3 area 11
  passive-interface default
  no passive-interface vlan 30
  exit
!
# Sécurité
enable password [NOM_2EME_BINOME]
service password-encryption
line console 0
  password [NOM_2EME_BINOME]
  login
  exit
```

### Configuration de SW1

```bash
enable
configure terminal
hostname SW1
!
vlan 10
  name DATA1
  exit
vlan 20
  name Management
  exit
!
# Port Access vers PC0
interface g0/0
  description "PC0 - VLAN 10"
  switchport mode access
  switchport access vlan 10
  spanning-tree portfast
  spanning-tree bpduguard enable
  no shutdown
  exit
!
# VLAN Management
interface vlan 20
  ip address 172.16.20.101 255.255.255.0
  no shutdown
  exit
!
# Trunks vers Fédérateurs
interface g0/1
  description "Trunk vers Fédérateur1"
  switchport mode trunk
  switchport trunk native vlan 20
  switchport trunk allowed vlan 10,15,20,30
  no shutdown
  exit
!
interface g0/2
  description "Trunk vers Fédérateur2"
  switchport mode trunk
  switchport trunk native vlan 20
  switchport trunk allowed vlan 10,15,20,30
  no shutdown
  exit
!
# Spanning Tree
spanning-tree mode rapid
spanning-tree portfast bpduguard default
!
# Sécurité
enable password [NOM_2EME_BINOME]
line console 0
  password [NOM_2EME_BINOME]
  login
  exit
```

### Configuration de SW2

```bash
enable
configure terminal
hostname SW2
!
vlan 10
  name DATA1
  exit
vlan 20
  name Management
  exit
!
# Port Access vers PC1
interface g0/0
  description "PC1 - VLAN 10"
  switchport mode access
  switchport access vlan 10
  spanning-tree portfast
  spanning-tree bpduguard enable
  no shutdown
  exit
!
# VLAN Management
interface vlan 20
  ip address 172.16.20.102 255.255.255.0
  no shutdown
  exit
!
# Trunks vers Fédérateurs
interface g0/1
  description "Trunk vers Fédérateur2"
  switchport mode trunk
  switchport trunk native vlan 20
  switchport trunk allowed vlan 10,15,20,30
  no shutdown
  exit
!
interface g0/2
  description "Trunk vers Fédérateur1"
  switchport mode trunk
  switchport trunk native vlan 20
  switchport trunk allowed vlan 10,15,20,30
  no shutdown
  exit
!
spanning-tree mode rapid
spanning-tree portfast bpduguard default
!
enable password [NOM_2EME_BINOME]
line console 0
  password [NOM_2EME_BINOME]
  login
  exit
```

### Configuration de CE11 (Siège1)

```bash
enable
configure terminal
hostname Siège1
!
# Loopback
interface loopback 0
  ip address 172.16.11.11 255.255.255.255
  no shutdown
  exit
!
# Interface WAN vers PE1
interface f0/0
  description "WAN vers PE1/Fédérateur1"
  ip address 192.168.1.2 255.255.255.252
  no shutdown
  exit
!
# OSPF
router ospf 300 vrf VPN_SSIR
  router-id 172.16.11.11
  network 192.168.1.0 0.0.0.3 area 11
  network 172.16.11.11 0.0.0.0 area 11
  log-adjacency-changes
  exit
!
# Route par défaut
ip route 0.0.0.0 0.0.0.0 192.168.1.1
!
# Sécurité
enable password [NOM_2EME_BINOME]
line console 0
  password [NOM_2EME_BINOME]
  login
  exit
line vty 0 4
  password [NOM_2EME_BINOME]
  login
  exit
```

### Configuration de CE21 (Siège2)

```bash
enable
configure terminal
hostname Siège2
!
# Loopback
interface loopback 0
  ip address 172.16.21.21 255.255.255.255
  no shutdown
  exit
!
# Interface WAN vers PE1
interface f0/0
  description "WAN vers PE1/Fédérateur1"
  ip address 192.168.1.6 255.255.255.252
  no shutdown
  exit
!
# OSPF - Area 21
router ospf 300 vrf VPN_SSIR
  router-id 172.16.21.21
  network 192.168.1.4 0.0.0.3 area 21
  network 172.16.21.21 0.0.0.0 area 21
  log-adjacency-changes
  exit
!
# Route par défaut
ip route 0.0.0.0 0.0.0.0 192.168.1.5
!
# Sécurité
enable password [NOM_2EME_BINOME]
line console 0
  password [NOM_2EME_BINOME]
  login
  exit
line vty 0 4
  password [NOM_2EME_BINOME]
  login
  exit
```

---

## Configuration des Branches

### Configuration de CE12 (Branch1)

```bash
enable
configure terminal
hostname Branch1
!
# Loopback
interface loopback 0
  ip address 172.16.12.12 255.255.255.255
  no shutdown
  exit
!
# Interface principale vers PE2
interface g1/0
  description "Lien principal vers PE2/SW3"
  no shutdown
  exit
!
# Subinterface 101 (Management)
interface g1/0.101
  encapsulation dot1Q 101
  description "VLAN 101 - Management"
  ip address 172.16.101.1 255.255.255.0
  no shutdown
  exit
!
# Subinterface 102 (DATA)
interface g1/0.102
  encapsulation dot1Q 102
  description "VLAN 102 - DATA"
  ip address 172.16.102.1 255.255.255.0
  no shutdown
  exit
!
# Subinterface 99 (WAN)
interface g1/0.99
  encapsulation dot1Q 99
  description "VLAN 99 - WAN vers PE2"
  ip address 192.168.1.10 255.255.255.252
  no shutdown
  exit
!
# DHCP pour VLAN 102
ip dhcp pool VLAN102_POOL
  network 172.16.102.0 255.255.255.0
  default-router 172.16.102.1
  dns-server 8.8.8.8 8.8.4.4
  lease 1 0
  exit
!
ip dhcp excluded-address 172.16.102.1 172.16.102.10
!
# OSPF - Area 12
router ospf 300 vrf VPN_SSIR
  router-id 172.16.12.12
  network 192.168.1.8 0.0.0.3 area 12
  network 172.16.101.0 0.0.0.255 area 12
  network 172.16.102.0 0.0.0.255 area 12
  log-adjacency-changes
  exit
!
# Route par défaut
ip route 0.0.0.0 0.0.0.0 192.168.1.9
!
# Sécurité
enable password [NOM_2EME_BINOME]
line console 0
  password [NOM_2EME_BINOME]
  login
  exit
line vty 0 4
  password [NOM_2EME_BINOME]
  login
  exit
```

### Configuration de SW3 (Branch1)

```bash
enable
configure terminal
hostname SW3
!
vlan 101
  name Management
  exit
vlan 102
  name DATA
  exit
!
# Port Access vers PC2
interface g0/0
  description "PC2 - VLAN 102 DATA"
  switchport mode access
  switchport access vlan 102
  spanning-tree portfast
  spanning-tree bpduguard enable
  no shutdown
  exit
!
# VLAN Management
interface vlan 101
  ip address 172.16.101.100 255.255.255.0
  no shutdown
  exit
!
# Trunk vers CE12
interface g0/1
  description "Trunk vers CE12"
  switchport mode trunk
  switchport trunk native vlan 101
  switchport trunk allowed vlan 101,102,99
  no shutdown
  exit
!
spanning-tree mode rapid
spanning-tree portfast bpduguard default
!
enable password [NOM_2EME_BINOME]
line console 0
  password [NOM_2EME_BINOME]
  login
  exit
```

### Configuration de CE22 (Branch2)

```bash
enable
configure terminal
hostname Branch2
!
# Loopback
interface loopback 0
  ip address 172.16.22.22 255.255.255.255
  no shutdown
  exit
!
# Interface principale
interface g1/0
  description "Lien principal vers PE2/SW4"
  no shutdown
  exit
!
# Subinterface 103 (Management)
interface g1/0.103
  encapsulation dot1Q 103
  description "VLAN 103 - Management"
  ip address 172.16.103.1 255.255.255.0
  no shutdown
  exit
!
# Subinterface 104 (DATA)
interface g1/0.104
  encapsulation dot1Q 104
  description "VLAN 104 - DATA"
  ip address 172.16.104.1 255.255.255.0
  no shutdown
  exit
!
# Subinterface 99 (WAN)
interface g1/0.99
  encapsulation dot1Q 99
  description "VLAN 99 - WAN vers PE2"
  ip address 192.168.1.14 255.255.255.252
  no shutdown
  exit
!
# DHCP pour VLAN 104
ip dhcp pool VLAN104_POOL
  network 172.16.104.0 255.255.255.0
  default-router 172.16.104.1
  dns-server 8.8.8.8 8.8.4.4
  lease 1 0
  exit
!
ip dhcp excluded-address 172.16.104.1 172.16.104.10
!
# OSPF - Area 22
router ospf 300 vrf VPN_SSIR
  router-id 172.16.22.22
  network 192.168.1.12 0.0.0.3 area 22
  network 172.16.103.0 0.0.0.255 area 22
  network 172.16.104.0 0.0.0.255 area 22
  log-adjacency-changes
  exit
!
# Route par défaut
ip route 0.0.0.0 0.0.0.0 192.168.1.13
!
# Sécurité
enable password [NOM_2EME_BINOME]
line console 0
  password [NOM_2EME_BINOME]
  login
  exit
line vty 0 4
  password [NOM_2EME_BINOME]
  login
  exit
```

### Configuration de SW4 (Branch2)

```bash
enable
configure terminal
hostname SW4
!
vlan 103
  name Management
  exit
vlan 104
  name DATA
  exit
!
# Port Access vers PC3
interface g0/0
  description "PC3 - VLAN 104 DATA"
  switchport mode access
  switchport access vlan 104
  spanning-tree portfast
  spanning-tree bpduguard enable
  no shutdown
  exit
!
# VLAN Management
interface vlan 103
  ip address 172.16.103.100 255.255.255.0
  no shutdown
  exit
!
# Trunk vers CE22
interface g0/1
  description "Trunk vers CE22"
  switchport mode trunk
  switchport trunk native vlan 103
  switchport trunk allowed vlan 103,104,99
  no shutdown
  exit
!
spanning-tree mode rapid
spanning-tree portfast bpduguard default
!
enable password [NOM_2EME_BINOME]
line console 0
  password [NOM_2EME_BINOME]
  login
  exit
```

### Vérifications HSRP

```bash
# Sur Fédérateur1 - État HSRP
Fédérateur1# show standby brief
Interface   Grp  Pri P State     Active Router   Standby Router
Vlan10      1    150 P Active    local           172.16.10.2
Vlan15      2    150 P Active    local           172.16.15.2
Vlan20      3    150 P Active    local           172.16.20.2
```

### Vérifications DHCP

```bash
# Sur Fédérateur1
Fédérateur1# show ip dhcp binding
IP address          Client-ID      Lease expiration        Type
172.16.10.50        0100.5056...   Feb 20 2025 03:45 AM   Automatic
172.16.15.60        0100.5056...   Feb 20 2025 02:15 AM   Automatic
```

---

## Dépannage Courant

### Problème: Les pings entre sites ne fonctionnent pas

**Solution:**
1. Vérifier les adjacences OSPF: `show ip ospf neighbor`
2. Vérifier les routes BGP: `show ip bgp vpnv4 all`
3. Vérifier les labels MPLS: `show mpls forwarding-table`
4. Vérifier la connectivité WAN: `ping` depuis chaque CE vers son PE

### Problème: HSRP ne bascule pas

**Solution:**
1. Vérifier l'état HSRP: `show standby`
2. Vérifier les timers: `show standby timers`
3. Vérifier la ligne de Fédérateur2: `show interface`

### Problème: DHCP ne fonctionne pas

**Solution:**
1. Vérifier les pools: `show ip dhcp pool`
2. Vérifier les exclusions: `show run | include excluded`
3. Vérifier que VRF est appliqué

---

**Configuration Complète pour Partie 2 - 2025**
