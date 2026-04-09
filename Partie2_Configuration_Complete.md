# PARTIE 2 : CONFIGURATION COMPLÈTE ET DÉTAILLÉE

## Table des Matières
1. [Introduction et Topologie](#introduction-et-topologie)
2. [Plan d'Adressage](#plan-dadressage)
3. [Configuration du Siège](#configuration-du-siège)
4. [Configuration des Branches](#configuration-des-branches)
5. [Configuration du Backbone MPLS](#configuration-du-backbone-mpls)
6. [Vérification et Tests](#vérification-et-tests)

---

## Introduction et Topologie

Ce document décrit la configuration complète d'un réseau MPLS/VPN avec :
- **1 Siège** avec redondance complète (2 Fédérateurs, 2 Switches Layer 2)
- **2 Branches** indépendantes (Branch1 et Branch2)
- **1 Backbone MPLS** avec 2 PE et 2 P

**Architecture:**
```
                    SIÈGE (Headquarters)
        ┌─────────────────────────────────────────┐
        │  Fédérateur1 ←→ Fédérateur2            │
        │   (HSRP/VRRP)  (EtherChannel)          │
        │       ↓              ↓                   │
        │    SW1, SW2      SW1, SW2               │
        │    PC0, PC1      PC0, PC1               │
        │   VLANs: 10,15,20,30                   │
        └─────────────────────────────────────────┘
                    ↓              ↓
                  CE11            CE21
                   (Siège1)       (Siège2)
                    ↓              ↓
        ┌─────────────────────────────────────┐
        │   BACKBONE MPLS (Area 0)            │
        │   PE1 ←→ P1 ←→ P2 ←→ PE2           │
        │   OSPF + MPLS + MP-BGP              │
        └─────────────────────────────────────┘
                    ↓              ↓
                  CE12            CE22
               (Branch1)        (Branch2)
                    ↓              ↓
        ┌──────────────┐  ┌──────────────┐
        │  SW3 + PC2   │  │  SW4 + PC3   │
        │ VLANs: 101,102 │ VLANs: 103,104 │
        └──────────────┘  └──────────────┘
```

---

## Plan d'Adressage

### Backbone MPLS
| Liaison | Adresse | Masque | Interfaces |
|---------|---------|--------|------------|
| PE1-P1 | 10.1.1.0 | /30 | g1/0 à g2/0 |
| PE1-P2 | 10.1.1.4 | /30 | g2/0 à g3/0 |
| PE2-P2 | 10.1.1.8 | /30 | g1/0 à g2/0 |
| PE2-P1 | 10.1.1.12 | /30 | g2/0 à g3/0 |
| P1-P2 | 10.1.1.20 | /30 | g1/0 à g1/0 |
| PE1 Loopback | 1.1.1.1 | /32 | Loopback 0 |
| PE2 Loopback | 2.2.2.2 | /32 | Loopback 0 |
| P1 Loopback | 3.3.3.3 | /32 | Loopback 0 |
| P2 Loopback | 4.4.4.4 | /32 | Loopback 0 |

### Siège
| Équipement | Interface | Adresse | VLAN |
|-----------|-----------|---------|------|
| Fédérateur1 | VLAN 10 | 172.16.10.1 | 10 (DATA1) |
| Fédérateur1 | VLAN 15 | 172.16.15.1 | 15 (DATA2) |
| Fédérateur1 | VLAN 20 | 172.16.20.1 | 20 (Mgmt) |
| Fédérateur1 | VLAN 30 | 192.168.1.1 | 30 (Trunk) |
| SW1 | VLAN 20 | 172.16.20.101 | 20 |
| SW2 | VLAN 20 | 172.16.20.102 | 20 |
| CE11 (Siège1) | g1/0 | 192.168.1.2 | WAN |
| CE11 | Loopback | 172.16.11.11 | - |
| CE21 (Siège2) | g1/0 | 192.168.1.6 | WAN |
| CE21 | Loopback | 172.16.21.21 | - |

### Branches
| Équipement | Interface | Adresse | VLAN |
|-----------|-----------|---------|------|
| CE12 (Branch1) | g0/0/0 | 192.168.1.10 | WAN |
| CE12 | g0/0/0.101 | 172.16.101.1 | 101 (Mgmt) |
| CE12 | g0/0/0.102 | 172.16.102.1 | 102 (DATA) |
| CE12 | Loopback | 172.16.12.12 | - |
| SW3 | VLAN 101 | 172.16.101.100 | 101 |
| CE22 (Branch2) | g0/0/0 | 192.168.1.14 | WAN |
| CE22 | g0/0/0.103 | 172.16.103.1 | 103 (Mgmt) |
| CE22 | g0/0/0.104 | 172.16.104.1 | 104 (DATA) |
| CE22 | Loopback | 172.16.22.22 | - |
| SW4 | VLAN 103 | 172.16.103.100 | 103 |

---

## Configuration du Siège

### Configuration de Fédérateur1 (Switch Layer 3 - HSRP Principal)

#### 1. Configuration Basique
```bash
enable
configure terminal
hostname Fédérateur1
!
# Configuration des VLANs
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
# Configuration des interfaces VLAN
interface vlan 10
  description "VLAN DATA1 - Siège"
  ip address 172.16.10.1 255.255.255.0
  standby 1 ip 172.16.10.254
  standby 1 priority 150
  standby 1 preempt
  standby 1 timers 3 10
  no shutdown
  exit
!
interface vlan 15
  description "VLAN DATA2 - Siège"
  ip address 172.16.15.1 255.255.255.0
  standby 2 ip 172.16.15.254
  standby 2 priority 150
  standby 2 preempt
  standby 2 timers 3 10
  no shutdown
  exit
!
interface vlan 20
  description "VLAN Management"
  ip address 172.16.20.1 255.255.255.0
  standby 3 ip 172.16.20.254
  standby 3 priority 150
  standby 3 preempt
  standby 3 timers 3 10
  no shutdown
  exit
!
interface vlan 30
  description "VLAN Trunk - Liaison vers Backbone MPLS"
  ip address 192.168.1.1 255.255.255.252
  no shutdown
  exit
```

#### 2. Configuration des Ports Access et Trunk
```bash
# Ports access vers Switches Layer 2
interface g1/0/1
  description "Port Access - SW1"
  switchport mode access
  switchport access vlan 10
  no shutdown
  exit
!
interface g1/0/2
  description "Port Access - SW2"
  switchport mode access
  switchport access vlan 10
  no shutdown
  exit
!
# Ports Trunk vers SW1 et SW2
interface g1/0/3
  description "Trunk vers SW1"
  switchport mode trunk
  switchport trunk native vlan 20
  switchport trunk allowed vlan 10,15,20,30
  no shutdown
  exit
!
interface g1/0/4
  description "Trunk vers SW2"
  switchport mode trunk
  switchport trunk native vlan 20
  switchport trunk allowed vlan 10,15,20,30
  no shutdown
  exit
!
# EtherChannel vers Fédérateur2 (Gig interfaces)
interface g1/0/5
  description "EtherChannel vers Fédérateur2 - Lien 1"
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
interface g1/0/6
  description "EtherChannel vers Fédérateur2 - Lien 2"
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

#### 3. Configuration DHCP
```bash
# DHCP pour VLAN 10 (DATA1)
ip dhcp pool VLAN10_POOL
  network 172.16.10.0 255.255.255.0
  default-router 172.16.10.254
  dns-server 8.8.8.8 8.8.4.4
  lease 1 0
  exit
!
ip dhcp excluded-address 172.16.10.1 172.16.10.10
!
# DHCP pour VLAN 15 (DATA2)
ip dhcp pool VLAN15_POOL
  network 172.16.15.0 255.255.255.0
  default-router 172.16.15.254
  dns-server 8.8.8.8 8.8.4.4
  lease 1 0
  exit
!
ip dhcp excluded-address 172.16.15.1 172.16.15.10
```

#### 4. Configuration OSPF (VRF VPN_SSIR)
```bash
# OSPF pour la liaison vers Backbone
ip vrf VPN_SSIR
  description "VPN Siège & Branches"
  rd 100:1
  route-target both 100:1
  exit
!
# Appliquer VRF à l'interface de liaison
interface vlan 30
  ip vrf forwarding VPN_SSIR
  ip address 192.168.1.1 255.255.255.252
  no shutdown
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
```

#### 5. Configuration Sécurité
```bash
# Mots de passe
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
enable password [NOM_2EME_BINOME]
service password-encryption
!
# Logs
logging buffered 1000
no logging console
```

### Configuration de Fédérateur2 (Switch Layer 3 - HSRP Secondaire)

```bash
enable
configure terminal
hostname Fédérateur2
!
# Configuration identique à Fédérateur1 SAUF:
# - Priority HSRP = 100 (non-primaire)
# - IP Management VLAN 20 = 172.16.20.2
!
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
  description "Liaison vers Backbone MPLS"
  ip vrf forwarding VPN_SSIR
  ip address 192.168.1.2 255.255.255.252
  no shutdown
  exit
!
# EtherChannel (configuration symétrique)
interface g1/0/5
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
interface g1/0/6
  switchport mode trunk
  switchport trunk allowed vlan 10,15,20,30
  channel-group 1 mode active
  no shutdown
  exit
!
# OSPF identique à Fédérateur1
router ospf 300 vrf VPN_SSIR
  network 172.16.10.0 0.0.0.255 area 11
  network 172.16.15.0 0.0.0.255 area 11
  network 192.168.1.0 0.0.0.3 area 11
  passive-interface default
  no passive-interface vlan 30
  exit
!
# Sécurité identique
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

### Configuration de SW1 (Switch Layer 2)

```bash
enable
configure terminal
hostname SW1
!
# VLANs
vlan 10
  name DATA1
  exit
vlan 20
  name Management
  exit
!
# Port access pour PC0 (VLAN 10)
interface g0/0
  description "PC0 - VLAN DATA1"
  switchport mode access
  switchport access vlan 10
  spanning-tree portfast
  no shutdown
  exit
!
# VLAN de Management
interface vlan 20
  description "Management SW1"
  ip address 172.16.20.101 255.255.255.0
  no shutdown
  exit
!
# Trunk vers Fédérateur1
interface g0/1
  description "Trunk vers Fédérateur1"
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
line vty 0 4
  password [NOM_2EME_BINOME]
  login
  exit
```

### Configuration de SW2 (Switch Layer 2)

```bash
enable
configure terminal
hostname SW2
!
# Configuration identique à SW1 SAUF:
# - IP Management = 172.16.20.102
# - Port pour PC1 peut être g0/0
!
vlan 10
  name DATA1
  exit
vlan 20
  name Management
  exit
!
interface g0/0
  description "PC1 - VLAN DATA1"
  switchport mode access
  switchport access vlan 10
  spanning-tree portfast
  no shutdown
  exit
!
interface vlan 20
  ip address 172.16.20.102 255.255.255.0
  no shutdown
  exit
!
interface g0/1
  description "Trunk vers Fédérateur2"
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

### Configuration de CE11 (Siège 1)

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
interface g1/0
  description "WAN vers Fédérateur1/PE1"
  ip address 192.168.1.2 255.255.255.252
  no shutdown
  exit
!
# OSPF pour le VPN
router ospf 300 vrf VPN_SSIR
  router-id 172.16.11.11
  network 192.168.1.0 0.0.0.3 area 11
  network 172.16.11.11 0.0.0.0 area 11
  log-adjacency-changes
  exit
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
!
# Configuration statique de la route par défaut (optionnel)
ip route 0.0.0.0 0.0.0.0 192.168.1.1
```

### Configuration de CE21 (Siège 2)

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
# Interface WAN vers Fédérateur2/PE1
interface g1/0
  description "WAN vers Fédérateur2/PE1"
  ip address 192.168.1.6 255.255.255.252
  no shutdown
  exit
!
# OSPF pour le VPN
router ospf 300 vrf VPN_SSIR
  router-id 172.16.21.21
  network 192.168.1.4 0.0.0.3 area 21
  network 172.16.21.21 0.0.0.0 area 21
  log-adjacency-changes
  exit
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
!
# Route par défaut
ip route 0.0.0.0 0.0.0.0 192.168.1.5
```

---

## Configuration des Branches

### Configuration de CE12 (Branch 1)

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
interface g0/0/0
  description "Lien principal vers PE2"
  no shutdown
  exit
!
# Subinterface 101 (Management)
interface g0/0/0.101
  encapsulation dot1Q 101
  description "VLAN 101 - Management Branch1"
  ip address 172.16.101.1 255.255.255.0
  no shutdown
  exit
!
# Subinterface 102 (DATA)
interface g0/0/0.102
  encapsulation dot1Q 102
  description "VLAN 102 - DATA Branch1"
  ip address 172.16.102.1 255.255.255.0
  no shutdown
  exit
!
# Subinterface WAN (pour PE2)
interface g0/0/0.99
  encapsulation dot1Q 99
  description "WAN vers PE2"
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
# OSPF pour le VPN
router ospf 300 vrf VPN_SSIR
  router-id 172.16.12.12
  network 192.168.1.8 0.0.0.3 area 12
  network 172.16.101.0 0.0.0.255 area 12
  network 172.16.102.0 0.0.0.255 area 12
  log-adjacency-changes
  exit
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
!
# Route par défaut
ip route 0.0.0.0 0.0.0.0 192.168.1.9
```

### Configuration de SW3 (Switch Branch 1)

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
# Port access pour PC2
interface g0/0
  description "PC2 - VLAN 102 DATA"
  switchport mode access
  switchport access vlan 102
  spanning-tree portfast
  no shutdown
  exit
!
# VLAN Management
interface vlan 101
  description "Management SW3"
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

### Configuration de CE22 (Branch 2)

```bash
enable
configure terminal
hostname Branch2
!
# Configuration identique à CE12 SAUF:
# - Loopback = 172.16.22.22/32
# - Subinterfaces = 103 (Mgmt) et 104 (DATA)
# - IP WAN = 192.168.1.14/30
# - OSPF Area 22
# - DHCP pour VLAN 104
!
interface loopback 0
  ip address 172.16.22.22 255.255.255.255
  no shutdown
  exit
!
interface g0/0/0
  description "Lien principal vers PE2"
  no shutdown
  exit
!
interface g0/0/0.103
  encapsulation dot1Q 103
  description "VLAN 103 - Management Branch2"
  ip address 172.16.103.1 255.255.255.0
  no shutdown
  exit
!
interface g0/0/0.104
  encapsulation dot1Q 104
  description "VLAN 104 - DATA Branch2"
  ip address 172.16.104.1 255.255.255.0
  no shutdown
  exit
!
interface g0/0/0.99
  encapsulation dot1Q 99
  description "WAN vers PE2"
  ip address 192.168.1.14 255.255.255.252
  no shutdown
  exit
!
ip dhcp pool VLAN104_POOL
  network 172.16.104.0 255.255.255.0
  default-router 172.16.104.1
  dns-server 8.8.8.8 8.8.4.4
  lease 1 0
  exit
!
ip dhcp excluded-address 172.16.104.1 172.16.104.10
!
router ospf 300 vrf VPN_SSIR
  router-id 172.16.22.22
  network 192.168.1.12 0.0.0.3 area 22
  network 172.16.103.0 0.0.0.255 area 22
  network 172.16.104.0 0.0.0.255 area 22
  log-adjacency-changes
  exit
!
enable password [NOM_2EME_BINOME]
line console 0
  password [NOM_2EME_BINOME]
  login
  exit
line vty 0 4
  password [NOM_2EME_BINOME]
  login
  exit
!
ip route 0.0.0.0 0.0.0.0 192.168.1.13
```

### Configuration de SW4 (Switch Branch 2)

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
interface g0/0
  description "PC3 - VLAN 104 DATA"
  switchport mode access
  switchport access vlan 104
  spanning-tree portfast
  no shutdown
  exit
!
interface vlan 103
  description "Management SW4"
  ip address 172.16.103.100 255.255.255.0
  no shutdown
  exit
!
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

---

## Configuration du Backbone MPLS

### Configuration de PE1 (Provider Edge 1)

```bash
enable
configure terminal
hostname PE1
!
# Loopback
interface loopback 0
  ip address 1.1.1.1 255.255.255.255
  no shutdown
  exit
!
# Interfaces Backbone
interface g1/0
  description "Lien vers P1"
  ip address 10.1.1.1 255.255.255.252
  no shutdown
  exit
!
interface g2/0
  description "Lien vers P2"
  ip address 10.1.1.5 255.255.255.252
  no shutdown
  exit
!
# Interfaces côté Clients
interface g3/0
  description "Lien vers CE11/Fédérateur1"
  no shutdown
  exit
!
interface g4/0
  description "Lien vers CE21/Fédérateur1"
  no shutdown
  exit
!
# Configuration CEF et MPLS
ip cef
mpls ip
mpls label protocol ldp
mpls ldp router-id loopback 0 force
!
# MPLS sur interfaces Backbone
interface g1/0
  mpls ip
  exit
!
interface g2/0
  mpls ip
  exit
!
# OSPF Backbone (Area 0)
router ospf 1
  router-id 1.1.1.1
  network 10.1.1.0 0.0.0.3 area 0
  network 10.1.1.4 0.0.0.3 area 0
  network 1.1.1.1 0.0.0.0 area 0
  log-adjacency-changes
  exit
!
# VRF VPN_SSIR
ip vrf VPN_SSIR
  description "VPN Siège et Branches"
  rd 100:1
  route-target both 100:1
  exit
!
# Appliquer VRF aux interfaces clients
interface g3/0
  ip vrf forwarding VPN_SSIR
  ip address 192.168.1.1 255.255.255.252
  no shutdown
  exit
!
interface g4/0
  ip vrf forwarding VPN_SSIR
  ip address 192.168.1.5 255.255.255.252
  no shutdown
  exit
!
# OSPF client (VRF)
router ospf 300 vrf VPN_SSIR
  router-id 1.1.1.1
  network 192.168.1.0 0.0.0.3 area 11
  network 192.168.1.4 0.0.0.3 area 21
  log-adjacency-changes
  redistribute bgp 100 subnets
  exit
!
# BGP/MP-BGP
router bgp 100
  no bgp default ipv4-unicast
  bgp router-id 1.1.1.1
  neighbor 2.2.2.2 remote-as 100
  neighbor 2.2.2.2 update-source loopback 0
  !
  # Address Family VPNv4
  address-family vpnv4 unicast
    neighbor 2.2.2.2 activate
    neighbor 2.2.2.2 send-community both
    exit-address-family
  !
  # Address Family VRF VPN_SSIR
  address-family ipv4 vrf VPN_SSIR
    redistribute ospf 300 vrf VPN_SSIR
    exit-address-family
  exit
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

### Configuration de PE2 (Provider Edge 2)

```bash
enable
configure terminal
hostname PE2
!
# Configuration identique à PE1 SAUF:
# - Loopback = 2.2.2.2/32
# - Interfaces WAN = g1/0 (P2) et g2/0 (P1) avec IPs différentes
# - Interfaces clients = g3/0 (CE12) et g4/0 (CE22)
# - IPs clients = 192.168.1.9/30 et 192.168.1.13/30
# - OSPF 300 pour Area 12 et 22
# - BGP neighbor = 1.1.1.1
!
interface loopback 0
  ip address 2.2.2.2 255.255.255.255
  no shutdown
  exit
!
interface g1/0
  description "Lien vers P2"
  ip address 10.1.1.9 255.255.255.252
  no shutdown
  exit
!
interface g2/0
  description "Lien vers P1"
  ip address 10.1.1.13 255.255.255.252
  no shutdown
  exit
!
interface g3/0
  description "Lien vers CE12"
  ip vrf forwarding VPN_SSIR
  ip address 192.168.1.9 255.255.255.252
  no shutdown
  exit
!
interface g4/0
  description "Lien vers CE22"
  ip vrf forwarding VPN_SSIR
  ip address 192.168.1.13 255.255.255.252
  no shutdown
  exit
!
ip cef
mpls ip
mpls label protocol ldp
mpls ldp router-id loopback 0 force
!
interface g1/0
  mpls ip
  exit
interface g2/0
  mpls ip
  exit
!
router ospf 1
  router-id 2.2.2.2
  network 10.1.1.8 0.0.0.3 area 0
  network 10.1.1.12 0.0.0.3 area 0
  network 2.2.2.2 0.0.0.0 area 0
  log-adjacency-changes
  exit
!
ip vrf VPN_SSIR
  rd 100:1
  route-target both 100:1
  exit
!
router ospf 300 vrf VPN_SSIR
  router-id 2.2.2.2
  network 192.168.1.8 0.0.0.3 area 12
  network 192.168.1.12 0.0.0.3 area 22
  log-adjacency-changes
  redistribute bgp 100 subnets
  exit
!
router bgp 100
  no bgp default ipv4-unicast
  bgp router-id 2.2.2.2
  neighbor 1.1.1.1 remote-as 100
  neighbor 1.1.1.1 update-source loopback 0
  !
  address-family vpnv4 unicast
    neighbor 1.1.1.1 activate
    neighbor 1.1.1.1 send-community both
    exit-address-family
  !
  address-family ipv4 vrf VPN_SSIR
    redistribute ospf 300 vrf VPN_SSIR
    exit-address-family
  exit
!
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

### Configuration de P1 (Provider 1)

```bash
enable
configure terminal
hostname P1
!
interface loopback 0
  ip address 3.3.3.3 255.255.255.255
  no shutdown
  exit
!
interface g1/0
  description "Lien vers P2"
  ip address 10.1.1.21 255.255.255.252
  no shutdown
  exit
!
interface g2/0
  description "Lien vers PE1"
  ip address 10.1.1.2 255.255.255.252
  no shutdown
  exit
!
interface g3/0
  description "Lien vers PE2"
  ip address 10.1.1.14 255.255.255.252
  no shutdown
  exit
!
# CEF et MPLS
ip cef
mpls ip
mpls label protocol ldp
mpls ldp router-id loopback 0 force
!
# MPLS sur toutes les interfaces
interface g1/0
  mpls ip
  exit
interface g2/0
  mpls ip
  exit
interface g3/0
  mpls ip
  exit
!
# OSPF Backbone
router ospf 1
  router-id 3.3.3.3
  network 10.1.1.20 0.0.0.3 area 0
  network 10.1.1.0 0.0.0.3 area 0
  network 10.1.1.12 0.0.0.3 area 0
  network 3.3.3.3 0.0.0.0 area 0
  log-adjacency-changes
  exit
!
# Sécurité
enable password [NOM_2EME_BINOME]
line console 0
  password [NOM_2EME_BINOME]
  login
  exit
```

### Configuration de P2 (Provider 2)

```bash
enable
configure terminal
hostname P2
!
interface loopback 0
  ip address 4.4.4.4 255.255.255.255
  no shutdown
  exit
!
interface g1/0
  description "Lien vers P1"
  ip address 10.1.1.22 255.255.255.252
  no shutdown
  exit
!
interface g2/0
  description "Lien vers PE2"
  ip address 10.1.1.10 255.255.255.252
  no shutdown
  exit
!
interface g3/0
  description "Lien vers PE1"
  ip address 10.1.1.6 255.255.255.252
  no shutdown
  exit
!
ip cef
mpls ip
mpls label protocol ldp
mpls ldp router-id loopback 0 force
!
interface g1/0
  mpls ip
  exit
interface g2/0
  mpls ip
  exit
interface g3/0
  mpls ip
  exit
!
router ospf 1
  router-id 4.4.4.4
  network 10.1.1.20 0.0.0.3 area 0
  network 10.1.1.8 0.0.0.3 area 0
  network 10.1.1.4 0.0.0.3 area 0
  network 4.4.4.4 0.0.0.0 area 0
  log-adjacency-changes
  exit
!
enable password [NOM_2EME_BINOME]
line console 0
  password [NOM_2EME_BINOME]
  login
  exit
```

---

## Vérification et Tests

### Vérifications OSPF

```bash
# Sur PE1 - Vérifier voisins OSPF Backbone
PE1# show ip ospf neighbor
Neighbor ID     Pri   State      Dead Time   Address         Interface
4.4.4.4           1   FULL/  -   00:00:39    10.1.1.6        GigabitEthernet2/0
3.3.3.3           1   FULL/  -   00:00:32    10.1.1.2        GigabitEthernet1/0
!
# Sur PE1 - Vérifier voisins OSPF VRF
PE1# show ip ospf neighbor 300 vrf VPN_SSIR
Neighbor ID     Pri   State      Dead Time   Address         Interface
172.16.11.11      1   FULL/BDR   00:00:31    192.168.1.2     GigabitEthernet3/0
172.16.21.21      1   FULL/  -   00:00:35    192.168.1.6     GigabitEthernet4/0
!
# Afficher la table OSPF
PE1# show ip route ospf
...
O       3.3.3.3 [110/2] via 10.1.1.2, 00:15:20, GigabitEthernet1/0
O       4.4.4.4 [110/2] via 10.1.1.6, 00:15:15, GigabitEthernet2/0
```

### Vérifications MPLS

```bash
# Vérifier les voisins LDP
PE1# show mpls ldp neighbor
Peer LDP Ident: 3.3.3.3:0; Local LDP Ident 1.1.1.1:0
  TCP connection: 3.3.3.3:646 - 1.1.1.1:15002
  State: Operational; Msgs sent/rcvd: 15/15; Downstream
  Up time: 00:10:25
!
# Afficher la LFIB (Label Forwarding Information Base)
PE1# show mpls forwarding-table
Local      Outgoing   Prefix           Outgoing Next Hop    Bytes Tag
Label      Label or VC Outgoing Tag    Interface           Switched
16         Pop Label  3.3.3.3/32       Gi1/0       10.1.1.2
17         Pop Label  4.4.4.4/32       Gi2/0       10.1.1.6
18         16         2.2.2.2/32       Gi1/0       10.1.1.2
!
# Vérifier les bindings de labels
PE1# show mpls ip binding
  3.3.3.3/32      [Incoming label: 16]
  4.4.4.4/32      [Incoming label: 17]
  2.2.2.2/32      [Incoming label: 18]
```

### Vérifications VRF et BGP

```bash
# Afficher les VRF
PE1# show ip vrf
  Name                             Default RD            Interfaces
  VPN_SSIR                         100:1                 Gi3/0, Gi4/0
!
# Afficher les routes BGP VPNv4
PE1# show ip bgp vpnv4 all summary
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
2.2.2.2         4   100      10      10        5    0    0 00:05:30        4
!
# Afficher les routes du VRF
PE1# show ip route vrf VPN_SSIR
Routing Table: VPN_SSIR
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic, H - nhrp, l - LISP
       + - replicated route, % - next hop override, p - overrides from PfxRcd
!
O       172.16.12.12/32 [110/2] via 192.168.1.2, 00:05:00, GigabitEthernet3/0
O       172.16.102.0/24 [110/3] via 192.168.1.2, 00:05:00, GigabitEthernet3/0
```

### Tests de Connectivité

```bash
# Depuis CE11 (Siège1)
CE11# ping 172.16.12.12
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.12.12, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 10/12/15 ms
!
CE11# ping 172.16.21.21
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 5/6/8 ms
!
CE11# ping 172.16.22.22
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 20/22/25 ms
!
# Depuis CE12 (Branch1)
CE12# ping 172.16.11.11
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 15/18/20 ms
!
CE12# ping 172.16.21.21
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 10/12/15 ms
```

### Vérification HSRP sur Fédérateurs

```bash
# Sur Fédérateur1 (priorité 150)
Fédérateur1# show standby brief
Interface   Grp  Pri P State     Active Router   Standby Router
Vlan10      1    150 P Active    local              172.16.10.2
Vlan15      2    150 P Active    local              172.16.15.2
Vlan20      3    150 P Active    local              172.16.20.2
!
# Sur Fédérateur2 (priorité 100)
Fédérateur2# show standby brief
Interface   Grp  Pri P State     Active Router   Standby Router
Vlan10      1    100   Standby   172.16.10.1    local
Vlan15      2    100   Standby   172.16.15.1    local
Vlan20      3    100   Standby   172.16.20.1    local
```

### Vérification DHCP

```bash
# Sur Fédérateur1
Fédérateur1# show ip dhcp binding
Bindings from all pools not associated with VRF:
IP address          Client-ID             Lease expiration        Type
172.16.10.50        0100.5056.a3b4.c6     Feb 20 2025 03:45 AM   Automatic
172.16.10.100       0100.5056.a3b4.c7     Feb 20 2025 04:30 AM   Automatic
172.16.15.60        0100.5056.a3b4.c8     Feb 20 2025 02:15 AM   Automatic
```

### Vérification des Interfaces

```bash
# Sur PE1
PE1# show interfaces summary
Interface                      IP-Address      Status                Protocol
GigabitEthernet1/0             10.1.1.1        up                    up
GigabitEthernet2/0             10.1.1.5        up                    up
GigabitEthernet3/0             192.168.1.1     up                    up
GigabitEthernet4/0             192.168.1.5     up                    up
Loopback0                       1.1.1.1         up                    up
!
# Vérifier les interfaces MPLS
PE1# show mpls interfaces
Interface              IP            Tunnel   BGP Static Operational
GigabitEthernet1/0     Yes (vrf)     No       No  No     Yes
GigabitEthernet2/0     Yes (vrf)     No       No  No     Yes
```

---

## Dépannage et Troubleshooting

### Problèmes Courants et Solutions

#### 1. OSPF ne converge pas
```bash
# Vérifier la configuration OSPF
show ip ospf process
show ip ospf interface
show ip ospf neighbor detail

# Vérifier les ACLs
show access-lists

# Réinitialiser OSPF
clear ip ospf process
```

#### 2. Les routes BGP/VPNv4 ne sont pas distribuées
```bash
# Vérifier la session BGP
show ip bgp summary
show ip bgp neighbors 2.2.2.2

# Vérifier les route-targets
show ip bgp vpnv4 all
show ip bgp vpnv4 vrf VPN_SSIR

# Vérifier les redistribution OSPF vers BGP
show run | include redistribute
```

#### 3. Pas de connectivité MPLS
```bash
# Vérifier les adjacences LDP
show mpls ldp neighbor
show mpls ldp neighbor detail

# Vérifier les labels
show mpls forwarding-table
show mpls ip binding

# Vérifier la configuration MPLS
show run | include mpls
```

#### 4. Problèmes HSRP/VRRP
```bash
# Vérifier l'état HSRP
show standby
show standby brief
show standby delay

# Vérifier les priorités
show run | include standby

# Debug HSRP
debug standby
```

#### 5. Problèmes DHCP
```bash
# Vérifier les pools DHCP
show ip dhcp pool
show ip dhcp binding
show ip dhcp conflict

# Vérifier les exclusions
show run | include excluded

# Debug DHCP
debug ip dhcp packet
```

### Commandes de Diagnostic Complètes

```bash
# Vérifier la connectivité de bout en bout
traceroute 172.16.22.22

# Vérifier les tables de routage complètes
show ip route summary
show ip route vrf VPN_SSIR summary

# Vérifier les statistiques d'interface
show interfaces statistics
show interfaces counters

# Vérifier les logs
show logging
show ip ospf events

# Diagnostiquer les problèmes de performance
show ip ospf statistics
show mpls ldp statistics

# Vérifier la configuration en cours
show running-config
show startup-config

# Comparer les configurations
show running-config | include router ospf
show running-config | include router bgp
```

---

## Notes Importantes

### Points Critiques à Vérifier

1. **Adressage**: Vérifier que tous les IPs sont corrects et uniques
2. **OSPF**: S'assurer que les areas sont bien configurées
3. **VRF**: Vérifier que les route-targets correspondent
4. **MPLS**: Tester les adjacences LDP avant les tests de bout en bout
5. **HSRP**: Vérifier les priorités et les timers
6. **DHCP**: S'assurer que les exclusions d'adresses sont correctes
7. **Sécurité**: Vérifier les mots de passe sur tous les équipements

### Conventions de Nommage

- **Hostnames**: 
  - Fédérateur1, Fédérateur2 (Switches Layer 3)
  - Siège1, Siège2 (Routeurs clients du Siège)
  - Branch1, Branch2 (Routeurs clients des Branches)
  - PE1, PE2 (Provider Edge)
  - P1, P2 (Provider)
  - SW1, SW2, SW3, SW4 (Switches Layer 2)

- **Mots de passe**:
  - Login: Nom du 1er binôme
  - Password: Nom du 2ème binôme

### Maintenance Régulière

```bash
# Archiver la configuration
copy running-config tftp://[IP_TFTP]/PE1-backup.cfg

# Vérifier les processus CPU
show processes cpu

# Vérifier la mémoire
show memory statistics

# Vérifier les logs d'erreur
show logging | include ERROR
```

---

**Document généré pour la configuration MPLS/VPN avec Siège et Branches**
**Date: 2025**
**Version: 1.0 - Configuration Complète**
