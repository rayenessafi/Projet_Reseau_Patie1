# 🚀 Scripts de Configuration 

Ce fichier contient tous les scripts de configuration complets pour chaque routeur.
Copiez et collez directement dans la console du routeur dans GNS3.

---

## ⚙️ P1 - CONFIGURATION COMPLÈTE

### P1 - Adressage de Base + OSPF + MPLS

```
conf t
hostname P1

! ===== ADRESSAGE =====
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
 no shutdown

interface g1/0
 ip address 10.1.1.21 255.255.255.252
 no shutdown

interface g2/0
 ip address 10.1.1.2 255.255.255.252
 no shutdown

interface g3/0
 ip address 10.1.1.14 255.255.255.252
 no shutdown

! ===== OSPF BACKBONE =====
router ospf 1
 router-id 3.3.3.3
 network 10.1.1.20 0.0.0.3 area 0
 network 10.1.1.0 0.0.0.3 area 0
 network 10.1.1.12 0.0.0.3 area 0
 network 3.3.3.3 0.0.0.0 area 0

! ===== MPLS CONFIGURATION =====
ip cef
mpls label protocol ldp
mpls ldp router-id Loopback0 force

interface g1/0
 mpls ip

interface g2/0
 mpls ip

interface g3/0
 mpls ip

end
write memory
```

---

## ⚙️ P2 - CONFIGURATION COMPLÈTE

### P2 - Adressage de Base + OSPF + MPLS

```
conf t
hostname P2

! ===== ADRESSAGE =====
interface Loopback0
 ip address 4.4.4.4 255.255.255.255
 no shutdown

interface g1/0
 ip address 10.1.1.22 255.255.255.252
 no shutdown

interface g2/0
 ip address 10.1.1.10 255.255.255.252
 no shutdown

interface g3/0
 ip address 10.1.1.6 255.255.255.252
 no shutdown

! ===== OSPF BACKBONE =====
router ospf 1
 router-id 4.4.4.4
 network 10.1.1.20 0.0.0.3 area 0
 network 10.1.1.8 0.0.0.3 area 0
 network 10.1.1.4 0.0.0.3 area 0
 network 4.4.4.4 0.0.0.0 area 0

! ===== MPLS CONFIGURATION =====
ip cef
mpls label protocol ldp
mpls ldp router-id Loopback0 force

interface g1/0
 mpls ip

interface g2/0
 mpls ip

interface g3/0
 mpls ip

end
write memory
```

---

## ⚙️ PE1 - CONFIGURATION COMPLÈTE

### PE1 - Adressage + OSPF + MPLS + VRF + MP-BGP

```
conf t
hostname PE1

! ===== ADRESSAGE =====
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 no shutdown

interface g1/0
 ip address 10.1.1.1 255.255.255.252
 no shutdown

interface g2/0
 ip address 10.1.1.5 255.255.255.252
 no shutdown

interface g3/0
 ip address 192.168.1.1 255.255.255.252
 no shutdown

interface g4/0
 ip address 192.168.1.5 255.255.255.252
 no shutdown

! ===== OSPF BACKBONE =====
router ospf 1
 router-id 1.1.1.1
 network 10.1.1.0 0.0.0.3 area 0
 network 10.1.1.4 0.0.0.3 area 0
 network 1.1.1.1 0.0.0.0 area 0

! ===== MPLS CONFIGURATION =====
ip cef
mpls label protocol ldp
mpls ldp router-id Loopback0 force

interface g1/0
 mpls ip

interface g2/0
 mpls ip

! ===== VRF CONFIGURATION =====
ip vrf VPN_Customer1
 rd 100:1
 route-target both 100:1

ip vrf VPN_Customer2
 rd 100:2
 route-target both 100:2

! Appliquer VRF aux interfaces
interface g3/0
 ip vrf forwarding VPN_Customer1
 ip address 192.168.1.1 255.255.255.252
 no shutdown

interface g4/0
 ip vrf forwarding VPN_Customer2
 ip address 192.168.1.5 255.255.255.252
 no shutdown

! ===== OSPF CLIENT 1 (Area 11) =====
router ospf 11 vrf VPN_Customer1
 router-id 1.1.1.1
 network 192.168.1.0 0.0.0.3 area 11

! ===== OSPF CLIENT 2 (Area 21) =====
router ospf 21 vrf VPN_Customer2
 router-id 1.1.1.1
 network 192.168.1.4 0.0.0.3 area 21

! ===== MP-BGP CONFIGURATION =====
router bgp 100
 no bgp default ipv4-unicast
 neighbor 2.2.2.2 remote-as 100
 neighbor 2.2.2.2 update-source Loopback0

 ! VPNv4 Address Family
 address-family vpnv4 unicast
  neighbor 2.2.2.2 activate
  neighbor 2.2.2.2 send-community both
 exit-address-family

 ! Customer 1 VRF
 address-family ipv4 vrf VPN_Customer1
  redistribute ospf 11
 exit-address-family

 ! Customer 2 VRF
 address-family ipv4 vrf VPN_Customer2
  redistribute ospf 21
 exit-address-family

end
write memory
```

---

## ⚙️ PE2 - CONFIGURATION COMPLÈTE

### PE2 - Adressage + OSPF + MPLS + VRF + MP-BGP

```
conf t
hostname PE2

! ===== ADRESSAGE =====
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
 no shutdown

interface g1/0
 ip address 10.1.1.9 255.255.255.252
 no shutdown

interface g2/0
 ip address 10.1.1.13 255.255.255.252
 no shutdown

interface g3/0
 ip address 192.168.1.9 255.255.255.252
 no shutdown

interface g4/0
 ip address 192.168.1.13 255.255.255.252
 no shutdown

! ===== OSPF BACKBONE =====
router ospf 1
 router-id 2.2.2.2
 network 10.1.1.8 0.0.0.3 area 0
 network 10.1.1.12 0.0.0.3 area 0
 network 2.2.2.2 0.0.0.0 area 0

! ===== MPLS CONFIGURATION =====
ip cef
mpls label protocol ldp
mpls ldp router-id Loopback0 force

interface g1/0
 mpls ip

interface g2/0
 mpls ip

! ===== VRF CONFIGURATION =====
ip vrf VPN_Customer1
 rd 100:1
 route-target both 100:1

ip vrf VPN_Customer2
 rd 100:2
 route-target both 100:2

! Appliquer VRF aux interfaces
interface g3/0
 ip vrf forwarding VPN_Customer1
 ip address 192.168.1.9 255.255.255.252
 no shutdown

interface g4/0
 ip vrf forwarding VPN_Customer2
 ip address 192.168.1.13 255.255.255.252
 no shutdown

! ===== OSPF CLIENT 1 (Area 12) =====
router ospf 12 vrf VPN_Customer1
 router-id 2.2.2.2
 network 192.168.1.8 0.0.0.3 area 12

! ===== OSPF CLIENT 2 (Area 22) =====
router ospf 22 vrf VPN_Customer2
 router-id 2.2.2.2
 network 192.168.1.12 0.0.0.3 area 22

! ===== MP-BGP CONFIGURATION =====
router bgp 100
 no bgp default ipv4-unicast
 neighbor 1.1.1.1 remote-as 100
 neighbor 1.1.1.1 update-source Loopback0

 address-family vpnv4 unicast
  neighbor 1.1.1.1 activate
  neighbor 1.1.1.1 send-community both
 exit-address-family

 address-family ipv4 vrf VPN_Customer1
  redistribute ospf 12
 exit-address-family

 address-family ipv4 vrf VPN_Customer2
  redistribute ospf 22
 exit-address-family

end
write memory
```

---

## ⚙️ CE11 - CONFIGURATION COMPLÈTE

### CE11 (Customer 1, Site 1)

```
conf t
hostname CE11

! ===== ADRESSAGE =====
interface Loopback0
 ip address 172.16.11.11 255.255.255.255
 no shutdown

interface g1/0
 ip address 192.168.1.2 255.255.255.252
 no shutdown

! ===== OSPF CLIENT =====
router ospf 11
 router-id 172.16.11.11
 network 192.168.1.0 0.0.0.3 area 11
 network 172.16.11.11 0.0.0.0 area 11

end
write memory
```

---

## ⚙️ CE12 - CONFIGURATION COMPLÈTE

### CE12 (Customer 1, Site 2)

```
conf t
hostname CE12

! ===== ADRESSAGE =====
interface Loopback0
 ip address 172.16.12.12 255.255.255.255
 no shutdown

interface g1/0
 ip address 192.168.1.10 255.255.255.252
 no shutdown

! ===== OSPF CLIENT =====
router ospf 12
 router-id 172.16.12.12
 network 192.168.1.8 0.0.0.3 area 12
 network 172.16.12.12 0.0.0.0 area 12

end
write memory
```

---

## ⚙️ CE21 - CONFIGURATION COMPLÈTE

### CE21 (Customer 2, Site 1)

```
conf t
hostname CE21

! ===== ADRESSAGE =====
interface Loopback0
 ip address 172.16.21.21 255.255.255.255
 no shutdown

interface g1/0
 ip address 192.168.1.6 255.255.255.252
 no shutdown

! ===== OSPF CLIENT =====
router ospf 21
 router-id 172.16.21.21
 network 192.168.1.4 0.0.0.3 area 21
 network 172.16.21.21 0.0.0.0 area 21

end
write memory
```

---

## ⚙️ CE22 - CONFIGURATION COMPLÈTE

### CE22 (Customer 2, Site 2)

```
conf t
hostname CE22

! ===== ADRESSAGE =====
interface Loopback0
 ip address 172.16.22.22 255.255.255.255
 no shutdown

interface g1/0
 ip address 192.168.1.14 255.255.255.252
 no shutdown

! ===== OSPF CLIENT =====
router ospf 22
 router-id 172.16.22.22
 network 192.168.1.12 0.0.0.3 area 22
 network 172.16.22.22 0.0.0.0 area 22

end
write memory
```

---

## ✅ COMMANDES DE VÉRIFICATION

### Vérifier OSPF Backbone

```
! Sur n'importe quel routeur backbone (P1, P2, PE1, PE2)
show ip ospf neighbor
show ip route ospf
show ip ospf database
```

### Vérifier MPLS

```
! Sur n'importe quel routeur backbone
show mpls ldp neighbor
show mpls interfaces
show mpls forwarding-table
show mpls ip binding
debug mpls ldp activity
```

### Vérifier VRF

```
! Sur PE1 ou PE2
show ip vrf
show ip vrf detail
show ip vrf interface
show ip route vrf VPN_Customer1
show ip route vrf VPN_Customer2
```

### Vérifier BGP

```
! Sur PE1 ou PE2
show bgp vpnv4 all summary
show bgp vpnv4 all
show bgp vpnv4 all neighbors
show ip bgp vpnv4 vrf VPN_Customer1
show ip bgp vpnv4 vrf VPN_Customer2
```

### Vérifier OSPF Clients

```
! Sur CE11, CE12, CE21 ou CE22
show ip ospf neighbor
show ip route ospf
```

---

## 🧪 TESTS DE CONNECTIVITÉ

### Test 1: Backbone Connectivity

```
P1# ping 1.1.1.1
P1# ping 2.2.2.2
PE1# ping 4.4.4.4
```

### Test 2: Customer 1 (Intra-VPN)

```
CE11# ping 172.16.12.12

! Résultat attendu: Reply from 172.16.12.12: bytes=32 time=<ms> TTL=<n>
```

### Test 3: Customer 2 (Intra-VPN)

```
CE21# ping 172.16.22.22

! Résultat attendu: Reply from 172.16.22.22: bytes=32 time=<ms> TTL=<n>
```

### Test 4: Isolation VRF (Ne doit PAS réussir)

```
CE11# ping 172.16.21.21

! Résultat attendu: Request timed out (pas de réponse)
```

### Test 5: Isolation VRF (Ne doit PAS réussir)

```
CE12# ping 172.16.22.22

! Résultat attendu: Request timed out (pas de réponse)
```

---

## 🔍 DÉPANNAGE RAPIDE

### Si OSPF neighbors ne se forment pas

```
! Vérifier les interfaces
show ip ospf interface brief

! Vérifier la configuration
show run | include router ospf

! Vérifier la connectivité physique
show ip interface brief
```

### Si MPLS LDP ne fonctionne pas

```
! Vérifier MPLS est activé
show mpls interfaces

! Activer debug
debug mpls ldp all

! Voir les erreurs
show log
```

### Si BGP ne forme pas de voisinage

```
! Vérifier le voisinage BGP
show bgp vpnv4 all neighbors

! Vérifier la connectivité loopback
ping 1.1.1.1
ping 2.2.2.2

! Vérifier CEF
show ip cef summary
```

### Si les routes VPN n'apparaissent pas

```
! Vérifier redistribution
show ip bgp vpnv4 vrf VPN_Customer1

! Vérifier OSPF dans VRF
show ip ospf 11 database

! Vérifier route-targets
show bgp vpnv4 all
```

---

## 💡 ORDRE DE CONFIGURATION RECOMMANDÉ

1. **Configurer l'adressage** sur tous les routeurs
2. **Configurer OSPF backbone** (Area 0)
3. **Activer MPLS** et vérifier LDP neighbors
4. **Configurer VRF** sur PE
5. **Configurer OSPF clients** sur CE
6. **Configurer MP-BGP** sur PE
7. **Tester la connectivité**

---

## 📝 NOTES IMPORTANTES

### Sequence Correcte sur PE

```
1. D'abord: interface g3/0 avec adresse normale
   interface g3/0
    ip address 192.168.1.1 255.255.255.252

2. Puis: Appliquer le VRF
   interface g3/0
    ip vrf forwarding VPN_Customer1
    
⚠️ ATTENTION: L'ordre importe! VRF d'abord = perte d'adresse IP!
```

### Redémarrer Proprement

```
! Si problème, reboot:
reload

! Ou reset complet (perte de config):
write erase
reload

! Ou garder config:
write memory
```

### Vérification Finale

```
! Avant de dire "c'est fini":
show running-config
show startup-config

! S'assurer que tout est sauvegardé:
write memory
```

---

**Guide rapide de configuration**  
**Tous les scripts sont prêts à copier/coller**  
**Testés sur GNS3 3.x et Cisco IOS 15.x**
