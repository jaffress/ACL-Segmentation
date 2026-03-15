# Segmentation & ACL

### Elif JAFFRES

---

## Introduction

Après avoir établi la haute disponibilité avec HSRP, ce second lab transforme l'infrastructure en une "Forteresse". L'objectif est d'appliquer les principes de segmentation et de filtrage (ACL) pour sécuriser les zones sensibles, en accord avec les recommandations de l'ANSSI.

---

## Étape A : Création des Zones

Nous isolons le réseau en trois segments logiques distincts. La passerelle par défaut pour chaque zone est portée par une IP virtuelle (HSRP) sur les switches de Niveau 3.

### 1. Définition des VLANs

Sur **Core-01** et **Core-02** :

```bash
vlan 10
 name GHOST_ADMIN  # Zone d'administration ultra-sensible
vlan 20
 name PROD_SERVEURS # Zone de stockage et services
vlan 30
 name USERS_CLIENTS # Zone utilisateurs (accès internet/bureautique)
```

### 2. Configuration des Interfaces Virtuelles (SVI)

On configure les adresses IP physiques de chaque switch avant d'activer le HSRP.

#### Exemple sur Core-01 :
```bash
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 standby 10 ip 192.168.10.254
 standby 10 priority 110
 standby 10 preempt

interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 standby 20 ip 192.168.20.254
 standby 20 priority 110
 standby 20 preempt

interface vlan 30
 ip address 192.168.30.1 255.255.255.0
 standby 30 ip 192.168.30.254
 standby 30 priority 110
 standby 30 preempt
```

### 3. Configuration du Switch d'Accès (Acces-01)
Comme ton infrastructure utilise un switch Acces-01 pour connecter les PCs, il faut mettre à jour l'attribution des ports car les VLANs ont changé (VLAN 20 est maintenant PROD, et VLAN 30 est USERS).

```bash
# Exemple : PC-PROD sur Fa0/1 et PC-USERS sur Fa0/2
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 20

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 30
```

---

## Étape B : Le Filtrage (ACL - Access Control List)

C'est ici qu'on transforme le réseau en "Forteresse". L'objectif est d'empêcher les utilisateurs du VLAN 30 d'accéder aux ressources sensibles du VLAN 10 (GHOST).

### 1. Création de l'ACL Étendue

Nous utilisons une **ACL Étendue** car elle permet de filtrer précisément par adresses IP source et destination.

```bash
# Sur Core-01 et Core-02 :
access-list 101 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 101 permit ip any any
```

**Explications des commandes :**
*   **access-list 101** : On utilise l'ID 101 (plage 100-199 pour les ACL étendues).
* **deny ip 192.168.30.0 0.0.0.255** : On interdit tout le trafic venant du réseau USERS (VLAN 30). Les "0.0.0.255" sont des masques génériques (wildcard masks).
* **192.168.10.0 0.0.0.255** : C'est la destination interdite (le réseau GHOST).
* **permit ip any any** : **CRITIQUE**. Par défaut, une ACL termine par un "tout interdire" (implicit deny). Sans cette ligne, les utilisateurs du VLAN 30 ne pourraient plus aller nulle part (ni en PROD, ni sur Internet).

### 2. Application de la Règle sur l'interface

Une règle écrite ne sert à rien si elle n'est pas appliquée. On l'applique sur l'interface virtuelle qui reçoit le trafic des utilisateurs.

```bash
interface vlan 30
 ip access-group 101 in
```

**Pourquoi "in" ?**
*   Le switch traite le trafic qui *entre* (in) depuis le VLAN 30. En bloquant dès l'entrée, on économise les ressources du processeur du switch qui n'aura pas à router un paquet destiné à être jeté plus tard.

---

## 💡 Note sur la Topologie

D'après ton image, tu as deux PCs. Pour tester la zone **GHOST**, je te conseille d'ajouter un troisième PC (PC-ADMIN) branché sur **Acces-01** et de le mettre dans le **VLAN 10**.

| Machines | VLAN | Ports | IPs |
| :--- | :--- | :--- | :--- |
| **PC-PROD** | 20 | Fa0/1 | 192.168.20.10 |
| **PC-USERS** | 30 | Fa0/2 | 192.168.30.10 |
| **PC-ADMIN** | 10 | Fa0/3 | 192.168.10.10 |

---

##  Tests de Sécurité & Validation

| Test | Action | Résultat Attendu | Statut |
| :--- | :--- | :--- | :---: |
| **Inter-VLAN (Classic)** | Ping USERS (VLAN 30) -> PROD (VLAN 20) | **Succès** (Routage OK) | ✅ |
| **Règle** | Ping USERS (VLAN 30) -> GHOST (VLAN 10) | **Échec** (Destination Unreachable) | 🔒 |
| **Admin Access** | Ping GHOST (VLAN 10) -> PROD (VLAN 20) | **Succès** (Admin libre) | ✅ |

> [!IMPORTANT]
> L'application d'une ACL sur une SVI filtre le trafic routé. Si les machines sont sur le même VLAN, l'ACL ne s'applique pas (commutation L2). La segmentation est donc la première ligne de défense indispensable.
