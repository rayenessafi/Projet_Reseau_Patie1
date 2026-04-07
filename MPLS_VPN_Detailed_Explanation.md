# 📖 Explication Détaillée du Projet IP/MPLS VPN

## Table des Matières
1. [Introduction au Projet](#introduction)
2. [Concepts Fondamentaux](#concepts)
3. [Architecture du Réseau](#architecture)
4. [Flux de Données](#flux-de-données)
5. [Technologies Utilisées](#technologies)
6. [Cas d'Usage Réel](#cas-dusage)

---

## <a name="introduction"></a>1. 🎯 Introduction au Projet

### Qu'est-ce qu'on va construire?

Ce projet implémente un **Backbone IP/MPLS multi-client** pour un opérateur télécommunications. C'est une infrastructure moderne qui permet à plusieurs clients de partager le même réseau sans pouvoir accéder au trafic les uns des autres.

### Analogie Simple

Imaginez un bâtiment avec plusieurs locataires:
- **Backbone MPLS** = Structure du bâtiment
- **Routeurs P** = Couloirs principaux
- **Routeurs PE** = Portes d'étages
- **VRF** = Appartements isolés
- **Clients CE** = Entrées des appartements

Chaque locataire a sa clé, peut accéder à son espace, mais ne peut pas accéder à celui des autres même si tous passent par les couloirs communs.

---

## <a name="concepts"></a>2. 📚 Concepts Fondamentaux

### A. MPLS (Multi Protocol Label Switching)

#### Qu'est-ce que c'est?

MPLS est une technologie qui remplace la recherche complexe d'adresses IP par de simples **étiquettes (labels)** de 20 bits. C'est comme marquer les paquets avec un numéro plutôt que de les router basé sur l'adresse IP.

#### Avantages de MPLS

| Avantage | Explication |
|----------|-------------|
| **Rapidité** | Recherche rapide du label plutôt que du préfixe IP |
| **Trafic Engineering** | Contrôle explicite des chemins |
| **QoS** | Champ EXP pour la qualité de service |
| **VPN** | Base pour les VPN MPLS |
| **Indépendance** | Fonctionne avec n'importe quel protocole couche 3 |

#### Comment fonctionne MPLS?

**Étape 1: Ingress LER (Label Edge Router)**
```
Paquet IP arrive → PE1 examine l'IP
                → Assigne Label 100
                → Envoie au LSR suivant
```

**Étape 2: LSR (Label Switch Router)**
```
Label 100 reçu → Consulte la table LIB
             → Label sortie = 200
             → Remplace label 100 par 200
             → Envoie au LSR suivant
```

**Étape 3: Egress LER**
```
Label 200 reçu → Dernière étape du LSP
              → Supprime le label
              → Remet le paquet IP original
              → Envoie au CE
```

#### Structure du Label MPLS

```
┌─────────────────────────────────────────┐
│  Label (20 bits)  │ EXP │ S │ TTL (8) │
├─────────────────────────────────────────┤
│ Identifiant local │QoS │Bottom│Hop Count│
└─────────────────────────────────────────┘

- Label: Identifie le flux unique entre deux routeurs adjacents
- EXP: 3 bits pour la classe de service (QoS)
- S: Bit de pile (indique si c'est le dernier label)
- TTL: Durée de vie (évite les boucles)
```

### B. VRF (Virtual Routing and Forwarding)

#### Qu'est-ce que c'est?

VRF est comme avoir plusieurs tables de routage indépendantes sur le même routeur. Chaque VRF a:
- Sa propre table de routage
- Ses propres interfaces
- Son propre espace d'adressage

#### Isolation VRF

```
Routeur PE1
├─ Global Routing Table (OSPF backbone)
├─ VRF VPN_Customer1
│  ├─ Tables de routage propres
│  ├─ Interfaces: Gi3/0 (vers CE11)
│  ├─ Routes OSPF Area 11
│  └─ Isolation complète
└─ VRF VPN_Customer2
   ├─ Tables de routage propres
   ├─ Interfaces: Gi4/0 (vers CE21)
   ├─ Routes OSPF Area 21
   └─ Isolation complète
```

#### Adressage Overlappé

VRF permet **deux clients d'utiliser la même adresse IP**:

```
Client 1: VRF VPN_Customer1
└─ 192.168.1.1 (PE1 Gi3/0)

Client 2: VRF VPN_Customer2
└─ 192.168.1.1 (PE1 Gi4/0)   ← Même adresse!

Ceci est possible car ils sont dans des VRF différentes!
```

### C. MP-BGP (Multiprotocol BGP)

#### Rôle de MP-BGP

MP-BGP est le **"postier"** du réseau. Son rôle:
1. Échanger les routes VPN entre PE
2. S'assurer que chaque PE sait où envoyer le trafic
3. Inclure l'étiquette MPLS à appliquer

#### BGP et VPNv4

```
Standard BGP: 
Routeur A → Routeur B
Préfixe IPv4: 192.168.1.0/24

MP-BGP avec VPNv4:
Routeur A → Routeur B
VPN-IPv4: 100:1:192.168.1.0/24
                ↑    ↑
            RD    IPv4
        (Route Distinguisher)

Le RD garantit l'unicité même avec adressage overlappé!
```

#### Route Distinguisher (RD) et Route Target (RT)

**Route Distinguisher (RD):**
- 8 octets = 2 octets Type + 6 octets Valeur
- Format: `ASN:Valeur` ou `IP:Valeur`
- Exemple: `100:1` = ASN 100, Valeur 1

**Route Target (RT):**
- Détermine quelles routes vont où
- Import: Routes à recevoir
- Export: Routes à envoyer
- Exemple: `route-target both 100:1` = Les routes avec RT 100:1

#### Exemple de Distribution

```
CE11 (172.16.11.11) → PE1 reçoit via OSPF Area 11
                    → Crée: RD 100:1, Route 172.16.11.11
                    → Ajoute RT 100:1
                    → Envoie à PE2 via BGP
                    
PE2 reçoit la route
 → Vérifie RT 100:1
 → Regarde sa table des RT
 → VRF VPN_Customer1 a "route-target import 100:1"
 → Ajoute la route à VRF VPN_Customer1
 → CE12 peut maintenant joindre CE11!
```

### D. OSPF (Open Shortest Path First)

#### Rôle de OSPF dans le projet

OSPF a **deux rôles distincts**:

**1. OSPF Backbone (Area 0):**
- Routage entre P et PE
- Échange les loopback addresses
- Utilisé pour établir le LSP MPLS

**2. OSPF Clients (Area 11, 12, 21, 22):**
- Routage entre CE
- Distribuées via BGP au PE distant
- Isolées dans les VRF

#### Hiérarchie OSPF du Projet

```
OSPF Instance 1 (Backbone):
└─ Area 0 (Backbone)
   ├─ Routeurs: P1, P2, PE1, PE2
   ├─ Réseaux: 10.1.1.0/24
   └─ Loopbacks: 1.1.1.1, 2.2.2.2, 3.3.3.3, 4.4.4.4

OSPF Instances Clients:
├─ Instance 11 (Area 11 - CE11 + PE1)
├─ Instance 12 (Area 12 - CE12 + PE2)
├─ Instance 21 (Area 21 - CE21 + PE1)
└─ Instance 22 (Area 22 - CE22 + PE2)
```

#### Configuration OSPF sur PE1

```
! Backbone OSPF
router ospf 1
 network 10.1.1.0 area 0      ← Backbone
 network 1.1.1.1 area 0

! Client 1 OSPF dans VRF
address-family ipv4 vrf VPN_Customer1
 redistribute ospf 11  ← Redistribue OSPF 11 vers BGP
```

---

## <a name="architecture"></a>3. 🏗️ Architecture du Réseau

### A. Vue Générale

```
                    BACKBONE IP/MPLS
                     Area 0 OSPF
                    10.1.1.0/24
              ┌─────────────────────┐
              │                     │
            P1(3.3.3.3)      P2(4.4.4.4)
           G2/0─────P─────G2/0
             │      |      │
             │      |      │
        G2/0 │      |      │ G3/0
             │    P─P     │
        G3/0 │      |      │ G1/0
             ▼      |      ▼
            PE1(1.1.1.1)  PE2(2.2.2.2)
         G3/0│ ├─────────┤ │G3/0
             │ │ BGP MP- │ │
        G4/0 │ │  VPNv4  │ │G4/0
             │ └─────────┘ │
        ┌────┴─┐        ┌──┴────┐
        │      │        │       │
       CE11   CE21     CE12    CE22
     (11.11) (21.21) (12.12)  (22.22)
       Cust1  Cust2    Cust1   Cust2
```

### B. Vue des VPN

```
VPN_Customer1 (Classe de service standard)
├─ CE11 (172.16.11.11) ─── PE1 ─┐
│                                 ├─ MPLS Tunnel
└─ CE12 (172.16.12.12) ─── PE2 ─┘
   Séparation complète VRF

VPN_Customer2 (Classe de service standard)
├─ CE21 (172.16.21.21) ─── PE1 ─┐
│                                 ├─ MPLS Tunnel  
└─ CE22 (172.16.22.22) ─── PE2 ─┘
   Séparation complète VRF
```

### C. Chemins MPLS (LSP)

```
CE11 (172.16.11.11) → PE1
│
├─ Crée paquet avec label L1
├─ Envoie à P1
│
P1 ← Reçoit label L1
│   ├─ Swaps: L1 → L2
│   └─ Envoie à P2
│
P2 ← Reçoit label L2
│   ├─ Swaps: L2 → L3
│   └─ Envoie à PE2
│
PE2 ← Reçoit label L3
    ├─ Pop: Retire L3
    └─ Regarde le paquet IP
       ├─ Destination: 172.16.12.12
       ├─ Cherche dans VRF VPN_Customer1
       └─ Envoie à CE12 directement
       
CE12 (172.16.12.12) reçoit le paquet!
```

---

## <a name="flux-de-données"></a>4. 🔄 Flux de Données Complets

### Scénario: CE11 ping CE12

#### Étape 1: Découverte des Routes

```
Jour 1 - Démarrage du réseau

1. CE11 émet: "Je suis 172.16.11.11"
   └─ OSPF Instance 11, Area 11

2. PE1 reçoit et stocke:
   └─ VRF VPN_Customer1: 172.16.11.11 via 192.168.1.2

3. PE1 via BGP crée:
   ├─ RD: 100:1
   ├─ VPN-IPv4: 100:1:172.16.11.11
   ├─ Route Target: 100:1
   └─ Label MPLS: 16 (exemple)

4. PE2 reçoit la route BGP:
   ├─ Vérifie Route Target: 100:1 ✓
   ├─ Match avec "route-target import 100:1" ✓
   ├─ Stocke dans VRF VPN_Customer1
   └─ Apprend: 172.16.11.11 via PE1 (1.1.1.1) Label 16

⚠️ DE MÊME:
CE12 émet, PE2 stocke, PE1 apprend via BGP
```

#### Étape 2: Préparation du Ping

```
CE11# ping 172.16.12.12

CE11 crée paquet ICMP:
├─ Source: 172.16.11.11
├─ Destination: 172.16.12.12
└─ Données: "Ping request"

CE11 encapsule pour envoyer à PE1:
├─ Destination MAC: PE1 Gi3/0
├─ Destination IP: 192.168.1.1
├─ Encapsule le paquet ICMP
└─ Envoie sur le réseau
```

#### Étape 3: Traitement chez PE1

```
PE1 reçoit sur Gi3/0:
├─ Interface dans VRF VPN_Customer1 ✓
├─ Paquet destiné à 172.16.12.12

PE1 consulte VRF VPN_Customer1:
├─ Destination: 172.16.12.12
├─ Route trouvée: via PE2, Label 16
├─ MPLS Label Stack: [Label 16 (pour PE2)]

PE1 crée l'en-tête MPLS:
├─ Label: 16 (label reçu de PE2)
├─ EXP: 0 (best effort)
├─ S: 1 (dernier label)
├─ TTL: 64

PE1 envoie à P1:
├─ PE1 Gi1/0 → P1 Gi2/0
├─ Avec label 16
└─ Paquet ICMP encapsulé
```

#### Étape 4: Traitement chez P1

```
P1 reçoit label 16:
├─ Consulte LFIB (Label Forwarding Information Base)
├─ Label 16: Swap to 17, Forward to P2

P1 Swap:
├─ Récupère le label 16
├─ Le remplace par 17
├─ TTL: 64 → 63
├─ Envoie à P2

P1 Gi2/0 → P2 Gi1/0 ← Avec label 17
```

#### Étape 5: Traitement chez P2

```
P2 reçoit label 17:
├─ Consulte LFIB
├─ Label 17: Swap to 18, Forward to PE2

P2 Swap:
├─ Label: 17 → 18
├─ TTL: 63 → 62
├─ Envoie à PE2

P2 Gi2/0 → PE2 Gi2/0 ← Avec label 18
```

#### Étape 6: Traitement chez PE2 (Egress)

```
PE2 reçoit label 18:
├─ Label 18: Pop (Penultimate Hop Popping)
├─ Label supprimé
├─ Paquet IP original exposé

PE2 regarde le paquet IP:
├─ Destination: 172.16.12.12
├─ Consulte VRF VPN_Customer1
├─ Interface de sortie: Gi3/0
├─ Route vers CE12: 192.168.1.10

PE2 route classiquement:
├─ Encapsule pour Gi3/0
├─ Destination MAC: CE12 Gi1/0
├─ Envoie sur Gi3/0

PE2 Gi3/0 → CE12 Gi1/0
└─ Paquet ICMP original
```

#### Étape 7: Réponse de CE12

```
CE12 reçoit le paquet ICMP:
├─ Source: 172.16.11.11
├─ Destination: 172.16.12.12

CE12 crée réponse:
├─ Source: 172.16.12.12
├─ Destination: 172.16.11.11
├─ Envoie à PE2

PE2 traite la réponse:
├─ Regarde VRF VPN_Customer1
├─ 172.16.11.11: Via PE1, Label 16
├─ Encapsule avec Label 16
├─ Envoie par LSP vers PE1

PE1 reçoit, desencapsule:
├─ Envoie à CE11
└─ CE11 reçoit la réponse!

✅ PING RÉUSSI!
```

### Isolation: Pourquoi CE11 ne peut pas joindre CE21

```
CE11# ping 172.16.21.21

CE11 crée paquet ICMP:
├─ Source: 172.16.11.11
├─ Destination: 172.16.21.21

CE11 envoie à PE1:
PE1 reçoit sur Gi3/0 (VRF VPN_Customer1):
├─ Cherche 172.16.21.21 dans VRF VPN_Customer1
├─ ❌ Pas trouvé! (172.16.21.21 est dans VRF VPN_Customer2)
├─ Senvoi pas le paquet
└─ Pas de route = Pas de livraison

❌ PING ÉCHOUE

Pourquoi?
- PE1 Gi4/0 (CE21) est dans VRF VPN_Customer2
- PE1 Gi3/0 (CE11) est dans VRF VPN_Customer1
- Les deux VRF ne partagent pas les routes
- Les routes de Customer2 ne sont pas dans VRF Customer1
```

---

## <a name="technologies"></a>5. 🔬 Approfondissement des Technologies

### A. Label Distribution Protocol (LDP)

#### Fonctionnement de LDP

```
Étape 1: Découverte des Voisins
┌─────────┐         UDP 646      ┌─────────┐
│  PE1    │────────Hello───────→│  PE2    │
│ 1.1.1.1 │←────────Hello───────│ 2.2.2.2 │
└─────────┘                       └─────────┘

Étape 2: Établissement de Session
- TCP 646 entre 1.1.1.1 et 2.2.2.2
- Échange des Router ID
- Négociation des paramètres

Étape 3: Distribution des Labels
PE2: "Le préfixe 2.2.2.2/32, le label est 16"
PE1: "OK, je vais utiliser 16 pour atteindre 2.2.2.2"
     "En retour, mon label pour 2.2.2.2 est 17"
PE2: "OK, je vais utiliser 17 pour retourner à PE1"
```

#### Label Binding

```
Label Binding = Association label ↔ Préfixe

Direction 1 (PE1 → PE2):
Préfixe 2.2.2.2/32 = Label 16 (PE1 à utiliser vers PE2)

Direction 2 (PE2 → PE1):
Préfixe 1.1.1.1/32 = Label 17 (PE2 à utiliser vers PE1)

Ces labels sont locaux entre les deux routeurs adjacents!
```

### B. FIB, LIB, et LFIB

#### Hiérarchie des Tables de Routage

```
┌──────────────────────────────────────────┐
│         Tables de Routage (FIB)          │
├──────────────────────────────────────────┤
│ Destination  │ Next Hop │ Interface │ Métrique
│ 10.1.1.0/30  │ Direct   │ Gi1/0     │ 1      │
│ 3.3.3.3/32   │ 10.1.1.2 │ Gi1/0     │ 2      │
│ 4.4.4.4/32   │ 10.1.1.2 │ Gi1/0     │ 3      │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│    Label Information Base (LIB)          │
├──────────────────────────────────────────┤
│ Destination  │ Received Label │ Route   │
│ 3.3.3.3/32   │ 16             │ Via P1  │
│ 4.4.4.4/32   │ 20             │ Via P1  │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│  Label Forwarding Information (LFIB)     │
├──────────────────────────────────────────┤
│ In Label │ Operation │ Out Label │ NH    │
│ 100      │ SWAP      │ 200       │ P1    │
│ 101      │ POP       │ -         │ P2    │
└──────────────────────────────────────────┘
```

### C. VPN-IPv4 et Route Distinguisher

#### Structure de VPN-IPv4

```
VPN-IPv4 = 96 bits (12 octets)

┌─────────────────────┬─────────────────┐
│ Route Distinguisher │   IPv4 Address  │
│   (8 octets)        │   (4 octets)    │
├─────────────────────┼─────────────────┤
│  100:1              │ 172.16.11.11/32 │
└─────────────────────┴─────────────────┘

Exemple complet:
100:1:172.16.11.11 
↑    ↑    ↑
AS   Val  IPv4 original

Cela garantit l'unicité car:
- 172.16.11.11 peut exister dans plusieurs VPN
- Mais 100:1:172.16.11.11 est unique dans le monde!
```

#### RD Types

```
Type 0: AS:VALUE
Format: ASN (2 octets):Valeur (4 octets)
Exemple: 100:1 (AS 100, Valeur 1)

Type 1: IP:VALUE  
Format: Adresse IP (4 octets):Valeur (2 octets)
Exemple: 192.168.1.1:1 (IP 192.168.1.1, Valeur 1)

Type 2: AS(4):VALUE (Extended AS)
Format: ASN (4 octets):Valeur (2 octets)
Exemple: 65100:1 (AS 65100, Valeur 1)
```

---

## <a name="cas-dusage"></a>6. 💼 Cas d'Usage Réel

### Scénario: Opérateur Télécommunications

#### Clients

**Client 1 (Banque ABC):**
- Siège: Paris (CE11)
- Succursale: Londres (CE12)
- Besoin: Réseau sécurisé et isolé
- Service: VPN_Customer1

**Client 2 (Entreprise XYZ):**
- Siège: Lyon (CE21)
- Bureau: Berlin (CE22)
- Besoin: Même réseau sécurisé
- Service: VPN_Customer2

#### Avantages de cette Solution

| Avantage | Bénéfice |
|----------|----------|
| **Isolation** | Banque n'accède jamais aux données de l'Entreprise |
| **Débit Garantie** | BGP + MPLS TE pour QoS |
| **Redondance** | Deux chemins via P1 et P2 |
| **Efficacité** | Partage du backbone entre clients |
| **Scalabilité** | Ajouter nouveaux clients facilement |

#### Flux de Trafic Type

```
Banque ABC (Client 1):
Paris (CE11) ←→ Londres (CE12)
└─ VPN_Customer1 isolé complètement
└─ Peut utiliser même adresse que Client 2
└─ Trafic protégé physiquement

Entreprise XYZ (Client 2):
Lyon (CE21) ←→ Berlin (CE22)
└─ VPN_Customer2 isolé complètement
└─ Même si adresse IP = Client 1
└─ Trafic protégé par VRF
```

#### Sécurité

**L'isolation VRF fournit:**

1. **Isolation Logique** - Tables de routage séparées
2. **Isolation Physique** - Pas d'interface partagée
3. **Isolation d'Adresses** - Adressage overlappé possible
4. **Isolation de Trafic** - BGP label distinct par VPN

**Ce qui est IMPOSSIBLE:**
```
❌ CE11 ne peut accéder à 172.16.21.21 (autre VPN)
❌ CE11 ne voit pas les routes de CE21
❌ Un administrateur doit configurer spécifiquement pour autoriser
❌ Les tables de routage sont complètement séparées
```

---

## Résumé Visuel: Chemin du Ping CE11→CE12

```
Step 1: Découverte         Step 2: Préparation         Step 3: En Route
────────────────         ────────────────────        ────────────
CE11: Je suis 11         CE11# ping 172.16.12.12    PE1: Paquet + Label 16
  ↓                        ↓                          ↓
PE1: Route vers 11      CE11: Crée paquet ICMP     P1: Label Swap 16→17
  ↓                        ↓                          ↓
BGP: 100:1:11           PE1: Encapsule pour 12     P2: Label Swap 17→18
  ↓                        ↓                          ↓
PE2: Apprend 11         PE1: Ajoute Label 16       PE2: Pop Label, Route
  ↓                        ↓                          ↓
✅ Routes échangées    ✅ Label décidé            ✅ Paquet livré


SÉCURITÉ: À chaque étape, les VRF isolent completement
──────────────────────────────────────────────────────
PE1 Gi3/0 (VPN_C1) ≠ PE1 Gi4/0 (VPN_C2)
PE2 Gi3/0 (VPN_C1) ≠ PE2 Gi4/0 (VPN_C2)
└─ Même si les addresses IP se chevauchent, pas de collision!
```

---

## Points Clés à Retenir

### MPLS
✅ Remplace IP lookup par simple label lookup  
✅ 20 bits label + QoS field (EXP)  
✅ Distribué par LDP ou RSVP  
✅ Fonctionne avec OSPF pour établir les LSP

### VRF
✅ Isolation logique complète  
✅ Tables de routage séparées  
✅ Permet adressage overlappé  
✅ Contrôle fin de qui accède à quoi

### MP-BGP
✅ Échange les routes VPN entre PE  
✅ Route Distinguisher garantit unicité  
✅ Route Target contrôle qui importe  
✅ Transport du label MPLS

### OSPF Dual Role
✅ Backbone (Area 0): PE↔P↔PE  
✅ Clients (Area 11-22): CE↔PE  
✅ Redistribué via BGP au PE distant

---

**Créé pour:** Guide Complet IP/MPLS VPN  
**Basé sur:** Cahier de charges SSIR  
**Niveau:** Intermédiaire à Avancé
