# Partie 4 — Workflow : Datacenter Spine-Leaf

**Bloc :** Datacenter TheBigOffice · **Outil :** Cisco Packet Tracer (Catalyst 3650) · **Certification :** CompTIA Network+

> Résumé en quelques mots : 
> 
> - 2 Spines / 2 Leafs compute / 2 Border Leafs 
> - Fabric `/30` `point-to-point` (pas de DR/BDR)
> - Border Leafs → **Core** sur `Gi1/0/3` & `Gi1/0/4` (ECMP N-S, `10.0.12/13.0`) 
> - VLAN 210 applicatif `172.16.2.0/24` / VLAN 220 data `172.16.3.0/24`
> - APP-WEB1/2 derrière VIP LB `.10`
> - SAN block store · sortie Internet via l'ASA (P3) + objet PAT `DC-NET`.
> 
> 5 incidents corrigés · 0 déviation · 3 limitations PT (LB, SAN, Jumbo MTU)
> 
> Plan d'adressage complet → [`IPAM.md`](../IPAM.md)

---
## Topologie As-Built

![Networ-overview-P4](../assets/network-overview/NO_P4.png)

---
## Niveaux & équipements

| Rôle            | Équipement                                              | Rôle dans la partie                                                                      |
| --------------- | ------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Spine (fabric)  | **DC-Spine1 / DC-Spine2** (3650) — *nouveaux*           | Cœur routé ; 4 downlinks routés chacun (2 Leafs + 2 Border Leafs) ; pas de VLAN/SVI      |
| Leaf compute    | **DC-Leaf1 / DC-Leaf2** (3650) — *nouveaux*             | Face serveurs ; passerelle SVI par tier ; 2 uplinks routés                               |
| Leaf border     | **DC-BorderLeaf1 / DC-BorderLeaf2** (3650) — *nouveaux* | Jonction N-S ; 2 uplinks routés (Spines) + 1 downlink routé (Core)                       |
| Tier applicatif | **APP-WEB1 `.11` / APP-WEB2 `.12`** — *nouveaux*        | Backend web interne (HTTP) ; sortie via ASA, **jamais en entrée**                        |
| Load balancer   | **LB-APP** (`.10` VIP) — *nouveau*                      | Présente la VIP applicative (round-robin conceptuel — dette PT)                          |
| Tier data       | **SAN-Server** (`.10` dans `172.16.3.0/24`) — *nouveau* | Rôle block-store ; joignable depuis le tier applicatif uniquement                        |
| Edge campus     | **CORE-SW** (3650)                                      | Gagne 2 liens `/30` routés vers les Border Leafs + 2 networks OSPF ; rien d'autre touché |

Tout le campus P1/P2/P3 est **inchangé** — P4 n'ajoute que la fabric et deux liens sur le Core. Le HQ-Router n'est pas touché (ses deux GigE déjà utilisés ; hors du chemin DC par conception).

---

## Étapes de configuration

La fabric se construit de l'intérieur : routage fabric (Spines → Leafs → Border Leafs), puis le raccord Core, puis les serveurs, puis la sortie. Le plan de contrôle (adjacences OSPF) est vérifié **avant** tout ping — un échec data-plane débogué à travers six équipements est le gouffre de temps classique.

```
[1] DC-Spine1 / DC-Spine2 : 4 downlinks routés /30 chacun, OSPF P2P
[2] DC-Leaf1 / DC-Leaf2   : 2 uplinks routés + VLAN/SVI + ports serveurs + OSPF (SVI passif)
[3] DC-BorderLeaf1 / BL2  : 2 uplinks routés (Spines) + 1 downlink routé (Core), OSPF P2P
[4] CORE-SW               : Gi1/0/3 + Gi1/0/4 routés vers BL1/BL2, 2 networks OSPF   <- le raccord N-S
[5] Serveurs              : APP-WEB1/2, LB-APP (VIP), SAN-Server — IP statique + HTTP
[6] Durcissement          : ports fabric inutilisés -> VLAN 998 + shutdown
[7] Edge (ASA, P3)        : route inside 172.16.2/3.0 + objet PAT DC-NET   <- sortie
```

**Câblage as-built** (subnets `/30` → [`IPAM.md §2`](../IPAM.md)) : Spine1 `Gi1/0/1–4` → Leaf1 / Leaf2 / BL1 / BL2 ; Spine2 `Gi1/0/1–4` → Leaf1 / Leaf2 / BL1 / BL2 ; BL1 `Gi1/0/3` ↔ **CORE `Gi1/0/3`** ; BL2 `Gi1/0/3` ↔ **CORE `Gi1/0/4`** ; Leaf1 `Gi1/0/5–7` → APP-WEB1/2 + LB (VLAN 210) ; Leaf2 `Gi1/0/5` → SAN (VLAN 220). Core `Gi1/0/1/2/5/24` intouchés.

---

### Étape 1 — DC-Spine1 & DC-Spine2 (fabric routée, pas de VLAN)

**Intention :** un Spine est du transit pur — chaque port `no switchport` + `/30` + `point-to-point`, aucun SVI.

```cisco
! ===== DC-Spine1 =====
enable
configure terminal
hostname DC-Spine1
ip routing
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.0.4.1 255.255.255.252
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 10.0.5.1 255.255.255.252
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet1/0/3
 no switchport
 ip address 10.0.8.1 255.255.255.252
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet1/0/4
 no switchport
 ip address 10.0.9.1 255.255.255.252
 ip ospf network point-to-point
 no shutdown
router ospf 1
 router-id 41.41.41.41
 network 10.0.4.0 0.0.0.3 area 0
 network 10.0.5.0 0.0.0.3 area 0
 network 10.0.8.0 0.0.0.3 area 0
 network 10.0.9.0 0.0.0.3 area 0
end
clear ip ospf process      ! confirmer "yes" pour appliquer le RID
write memory
```

**DC-Spine2 =** identique, RID `42.42.42.42`, adresses `10.0.6.1 / 10.0.7.1 / 10.0.10.1 / 10.0.11.1`.

**Validation :** `show ip interface brief` → les quatre `Gi1/0/1–4` `up/up`. (Les voisins montent à l'étape 3.)

---

### Étape 2 — DC-Leaf1 (VLAN 210, applicatif) & DC-Leaf2 (VLAN 220, data)

**Intention :** l'ordre est porteur (SVI down/down). Créer le VLAN → mettre un port serveur dedans → le serveur monte le lien → **alors** le SVI passe `up/up`. Un SVI sans port up dans son VLAN reste `down/down` et n'est jamais annoncé.

```cisco
! ===== DC-Leaf1 — tier APPLICATIF =====
enable
configure terminal
hostname DC-Leaf1
ip routing
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.0.4.2 255.255.255.252
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 10.0.6.2 255.255.255.252
 ip ospf network point-to-point
 no shutdown
vlan 210
 name DC-SERVERS
exit
interface vlan 210
 ip address 172.16.2.1 255.255.255.0
 no shutdown
interface range GigabitEthernet1/0/5 - 7
 switchport mode access
 switchport access vlan 210
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
router ospf 1
 router-id 43.43.43.43
 passive-interface default
 no passive-interface GigabitEthernet1/0/1
 no passive-interface GigabitEthernet1/0/2
 network 10.0.4.0 0.0.0.3 area 0
 network 10.0.6.0 0.0.0.3 area 0
 network 172.16.2.0 0.0.0.255 area 0
end
clear ip ospf process
write memory
```

```cisco
! ===== DC-Leaf2 — tier DATA =====
enable
configure terminal
hostname DC-Leaf2
ip routing
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.0.5.2 255.255.255.252
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 10.0.7.2 255.255.255.252
 ip ospf network point-to-point
 no shutdown
vlan 220
 name DC-STORAGE
exit
interface vlan 220
 ip address 172.16.3.1 255.255.255.0
 no shutdown
interface GigabitEthernet1/0/5
 switchport mode access
 switchport access vlan 220
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
router ospf 1
 router-id 44.44.44.44
 passive-interface default
 no passive-interface GigabitEthernet1/0/1
 no passive-interface GigabitEthernet1/0/2
 network 10.0.5.0 0.0.0.3 area 0
 network 10.0.7.0 0.0.0.3 area 0
 network 172.16.3.0 0.0.0.255 area 0
end
clear ip ospf process
write memory
```

**Validation :** `show ip interface brief | include Vlan210` → `172.16.2.1 up up` (Leaf1) ; idem `Vlan220` sur Leaf2. Le SVI est **annoncé mais passif** — aucun voisin ne se forme sur le VLAN serveur.

---

### Étape 3 — DC-BorderLeaf1 & DC-BorderLeaf2 (jonction N-S)

**Intention :** trois liens routés chacun (Spine1, Spine2, Core). Pas de VLAN serveur.

```cisco
! ===== DC-BorderLeaf1 =====
enable
configure terminal
hostname DC-BorderLeaf1
ip routing
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.0.8.2 255.255.255.252     ! Spine1
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet1/0/2
 no switchport
 ip address 10.0.10.2 255.255.255.252    ! Spine2
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet1/0/3
 no switchport
 ip address 10.0.12.1 255.255.255.252    ! Core
 ip ospf network point-to-point
 no shutdown
router ospf 1
 router-id 45.45.45.45
 network 10.0.8.0 0.0.0.3 area 0
 network 10.0.10.0 0.0.0.3 area 0
 network 10.0.12.0 0.0.0.3 area 0
end
clear ip ospf process
write memory
```

**DC-BorderLeaf2 =** identique, RID `46.46.46.46`, adresses `10.0.9.2` (Spine1) / `10.0.11.2` (Spine2) / `10.0.13.1` (Core).

**Validation (c'est ici que la fabric se prouve) :**
- Sur chaque **Spine** — `show ip ospf neighbor` → **4** voisins, tous `FULL/ -`. [P-01](#p-01)
- Sur chaque **Border Leaf** — **3** voisins (2 Spines + Core, une fois l'étape 4 faite), tous `FULL/ -` ; `show ip route 172.16.2.0` = ECMP via les deux Spines. [P-03](#p-03)
- Sur chaque **Leaf** — **2** voisins, tous `FULL/ -`. [P-04](#p-04)
- Partout — la colonne State est `FULL/  -` (**tiret**, jamais `FULL/DR`/`FULL/BDR`). Le tiret est la preuve du `point-to-point`.

---

### Étape 4 — CORE-SW (le raccord Nord-Sud — seul équipement campus touché)

**Intention :** ajouter deux ports routés + deux networks OSPF. Ne **pas** recréer de SVI `.1` sur le Core.

```cisco
! ===== CORE-SW (ajout uniquement) =====
enable
configure terminal
interface GigabitEthernet1/0/3
 no switchport
 ip address 10.0.12.2 255.255.255.252     ! BorderLeaf1
 ip ospf network point-to-point
 no shutdown
interface GigabitEthernet1/0/4
 no switchport
 ip address 10.0.13.2 255.255.255.252     ! BorderLeaf2
 ip ospf network point-to-point
 no shutdown
router ospf 1
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.13.0 0.0.0.3 area 0
end
write memory
```

**Validation :**
- `show ip ospf neighbor` (Core) → **5** voisins : `2.2.2.2` (DIST1), `3.3.3.3` (DIST2), `45.45.45.45` (BL1), `46.46.46.46` (BL2), `4.4.4.4` (HQ) — tous `FULL/ -`. [P-05](#p-05)
- `show ip route 172.16.2.0` (Core) → **deux descripteurs**, `via 10.0.12.1` et `via 10.0.13.1`, métrique égale → **ECMP N-S prouvé**. [P-06](#p-06)
- Sur **HQ-Router** — `show ip route ospf | include 172.16` → `O 172.16.2.0` et `O 172.16.3.0` via `10.0.1.1` → le DC a atteint l'edge. [P-07](#p-07)

---

### Étape 5 — Serveurs & Load Balancer

**Intention :** IP statique sur chaque `Server-PT`, HTTP activé, `index.html` marqué. Plan d'adressage → [`IPAM.md`](../IPAM.md) ; les marqueurs ci-dessous sont propres à la reproduction.

| Serveur | IP | VLAN | Marqueur `index.html` |
|---|---|---|---|
| APP-WEB1 | `172.16.2.11` | 210 | « TheBigOffice Web-App 1 » |
| APP-WEB2 | `172.16.2.12` | 210 | « TheBigOffice Web-App 2 » |
| LB-APP | `172.16.2.10` (VIP) | 210 | « Load Balancer — VIP / pool / round-robin / health-check » |
| SAN-Server | `172.16.3.10` | 220 | (HTTP ou FTP, endpoint testable) |

> **Dette LB (la consigner) :** PT n'a pas de moteur de load-balancer. `LB-APP` porte la VIP et *affiche* le round-robin/health-check qu'il exécuterait en prod, mais `http://172.16.2.10` atteint la propre page du LB, pas un `.11`/`.12` réparti. Même classe que « source SPAN ignorée » de P3. Prod = HAProxy / F5 / Nginx.

**Validation :** `http://172.16.2.11` = « Web-App 1 » [P-14](#p-14), `http://172.16.2.12` = « Web-App 2 » [P-15](#p-15), `http://172.16.2.10` = page LB [P-16](#p-16).

---

### Étape 6 — Durcissement

**Intention :** parquer les ports GigE inutilisés sur **chaque** Spine / Leaf / Border Leaf.

```cisco
configure terminal
vlan 998
 name QUARANTINE
exit
interface range GigabitEthernet1/0/8 - 24
 switchport mode access
 switchport access vlan 998
 shutdown
end
write memory
```

*(Sur les Spines/BLs sans port serveur, commencer la plage à `Gi1/0/5`.)*

**Validation :** `show interface status` → ports inutilisés `disabled` VLAN 998 ; ports routés `routed` ; ports serveurs `connected`. [P-09](#p-09)

---

### Étape 7 — Edge (ASA, P3) — joignabilité DC + sortie

**Intention :** l'ASA a besoin de la route DC (la statique P3 qui n'avait pas de destination) et d'un objet PAT (le PAT `192.168.0.0/16` ne couvre **pas** le DC).

```cisco
! ASA-EDGE
configure terminal
! -- joignabilité vers le DC --
route inside 172.16.2.0 255.255.255.0 192.168.200.2
route inside 172.16.3.0 255.255.255.0 192.168.200.2
! -- traduire le tier applicatif en sortie --
object network DC-NET
 subnet 172.16.2.0 255.255.254.0        ! /23 couvre .2 et .3
 nat (inside,outside) dynamic interface
end
```

**Validation :**
- `show route` → `172.16.0.0/24 … 3 subnets` : le `/24` DMZ connecté + `S 172.16.2.0` + `S 172.16.3.0` via `192.168.200.2`. [P-20](#p-20)
- Depuis **APP-WEB1** — `ping 8.8.8.8` → `TTL=249` (après 1–3 timeouts de build). [P-13](#p-13)
- `show nat` → `DC-NET translate_hits ≠ 0` (session : `translate_hits = 3, untranslate_hits = 2`). **Ce compteur est la preuve durable de la sortie.** [P-12](#p-12)
- `show xlate` montre l'entrée PAT `DC-NET` **seulement tant que le flux est vivant** (timeout ICMP 30 s). Ne pas lire une table vieillie comme un échec ; lire le compteur. [P-21](#p-21)

---

## Validation de bout en bout (gate final)

| Couche       | Commande clé                                | Attendu                                                                | Preuve                         |
| ------------ | ------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------ |
| OSPF         | `show ip ospf neighbor`                     | Spine ×4 / BL ×3 / Leaf ×2 / Core ×5, tous `FULL/ -`, pas de DR/BDR    | [P-01](#p-01), [P-03](#p-03), [P-04](#p-04), [P-05](#p-05) |
| ECMP         | `show ip route 172.16.2.0` (Core)           | deux next-hops : `10.0.12.1` + `10.0.13.1`                             | [P-06](#p-06)                         |
| Edge         | `show ip route ospf \| include 172.16` (HQ) | `O 172.16.2.0` + `O 172.16.3.0`                                        | [P-07](#p-07)                         |
| Edge (ASA)   | `show route` (ASA)                          | `3 subnets`, `S 172.16.2.0` + `S 172.16.3.0`                           | [P-20](#p-20)                         |
| SVI          | `show ip interface brief \| include Vlan21` | `172.16.2.1` / `172.16.3.1` `up/up`                                    | [P-09](#p-09)                         |
| E-O          | APP-WEB1 `ping 172.16.3.10`                 | 4/4 (2e run)                                                           | [P-10](#p-10)                         |
| N-S          | PC campus `ping 172.16.2.11` + `tracert`    | 4/4 ; chemin `DIST→Core→BL→Spine→Leaf1`                                | [P-11](#p-11)                         |
| Sortie       | APP-WEB1 `ping 8.8.8.8` + `show nat`        | réponse ; `DC-NET translate_hits ≠ 0`                                  | [P-13](#p-13), [P-12](#p-12)                 |
| Isolation    | `show access-list DMZ-RESTRICT` (ASA)       | ligne 5 `deny ip host 172.16.0.10 any` hitcnt monte sur WEB-PUBLIC→APP | [P-17](#p-17)                         |
| Durcissement | `show interface status`                     | ports inutilisés `disabled` VLAN 998                                   | [P-09](#p-09)                         |

> Un `Request timed out` en tête (jusqu'à 3 sur les plus longs flux N-S/sortie) sur le **premier** paquet d'un flux frais est un build ARP + xlate/CEF, **pas** une faute. Un flux *bloqué* n'est prouvé que par le **compteur** de la ligne concernée. Les ports d'accès campus qui flashent ambre puis vert, avec 0 % de perte, sont une reconvergence STP cosmétique (PortFast confirmé), classée artefact PT.

---

## Dépannage (incidents de session)

> Incidents attrapés et corrigés le jour même — **pas** des dettes.

| # | Symptôme | Cause racine | Diagnostic | Correctif |
|---|---|---|---|---|
| 1 | APP-WEB1 sans Internet ; campus↔DC OK | **L'ASA n'avait aucune route vers `172.16.2/3.0`** — les statiques P3 n'avaient jamais été tapées (pas de destination jusqu'à P4) | `show route inside` = `172.16.0.0/24 … 1 subnets` (DMZ seul), pas 3 | `route inside 172.16.2.0/24` + `172.16.3.0/24 → 192.168.200.2` |
| 2 | La sortie échoue encore après la route | **DC non couvert par le NAT** — `OBJ-INSIDE-PAT` est `192.168.0.0/16` ; le serveur sortait non traduit | `show nat` / `show run object` — aucun objet ne matchait `172.16.2.x` | `object network DC-NET subnet 172.16.2.0/23` PAT dynamique |
| 3 | `show run interface Gi1/0/5` → `% Invalid input` | Commande lancée depuis `(config)#` | le marqueur `^` | `end` puis `show running-config interface …`, ou `do show …` — [P-18](#p-18) |
| 4 | Ports d'accès campus flashent ambre → vert | Reconvergence STP cosmétique (TCN / jitter PT) — **pas** un PortFast manquant (`… fa0/3 detail` → « portfast mode ») | 0 % de perte partout ; `transitions to forwarding: 1` | Artefact PT, non combattu — [P-19](#p-19) |
| 5 | Les 1–3 premiers paquets d'un flux N-S / sortie timeout | Build ARP + xlate/CEF sur plusieurs sauts | réponse au paquet 2–4 avec le bon TTL ; runs suivants 4/4 | Attendu — verdict lu sur les paquets suivants — [P-10](#p-10) |

---

## 5. Registre d'erreurs & dette technique (état final) 

> État final de chaque point. Les lignes sans `#` sont des décisions/observations locales à P4 (non référencées ailleurs) ; `#15` est l'identifiant global porté.

| # | Point | Domaine | Statut |
|---|---|---|---|
| — | SVI down/down si aucun port up dans son VLAN | 🟢 L2/L3 | ✅ Évité — port serveur affecté avant validation |
| — | Ports inutilisés ouverts | 🟢 Durcissement | ✅ VLAN 998 + shutdown sur tous les switches de fabric |
| LB | Round-robin / health-check du load balancer | 🟠 Haute disponibilité | 📋 Dette PT — VIP présente, logique conceptuelle (prod HAProxy/F5) |
| SAN | Rôle block-store | 🟠 Services | 📋 Dette PT — seule une réponse ping/HTTP |
| DMZ→APP | Reverse-proxy WEB-PUBLIC→APP bloqué | 🟠 Sécurité | 📋 Dépendance P3 — ajouter `permit tcp host 172.16.0.10 172.16.2.0/24 eq www` au-dessus de `DMZ-RESTRICT` ligne 5 |
| — | `/30` au lieu de `/31` | 🟢 Adressage | 📋 lisibilité pédagogique |
| — | Aire 0 unique | 🟠 Routage | 📋 accepté à l'échelle |
| — | Une passerelle SVI par leaf (SPOF) | 🟠 Haute disponibilité | 🔜 prod = Anycast Gateway (VXLAN EVPN), hors PT |
| — | MTU Jumbo 9000 | 🟢 Performance | 📋 documenté, rejeté par PT |
| 15 | Core = transit L3 N-S unique | 🟠 Haute disponibilité | 📋 porté — l'ECMP masque une panne de lien, pas une panne du Core |

---

## Annexe — Captures de preuve

> Une capture **canonique** par affirmation ; jumeaux périmés/dupliqués écartés. Le gate cite le `[P-##]` pertinent. Embeds Obsidian. *(Numérotation : `[P-02]` et `[P-08]` fusionnés dans `[P-03]` au triage — non attribués.)*

**<a id="p-01"></a> [P-01] · Adjacences Spine** — DC-Spine1 `show ip ospf neighbor`, 4× `FULL/ -`

![Capture P4-24](../assets/captures/P4/Capture_P4_24.png)

**<a id="p-03"></a> [P-03] · Adjacences Border Leaf + ECMP fabric** — DC-BorderLeaf2, 3 voisins, puis `route 172.16.2.0` via 10.0.9.1 + 10.0.11.1

![Capture P4-20](../assets/captures/P4/Capture_P4_20.png)

**<a id="p-04"></a> [P-04] · Adjacences Leaf** — DC-Leaf1, 2× `FULL/ -`

![Capture P4-01](../assets/captures/P4/Capture_P4_01.png)

**<a id="p-05"></a> [P-05] · Adjacences Core** — CORE-SW, 5 voisins (DIST1/2, BL1/2, HQ)

![Capture P4-23](../assets/captures/P4/Capture_P4_23.png)

**<a id="p-06"></a> [P-06] · ECMP N-S au Core** — CORE-SW `route 172.16.2.0` via 10.0.12.1 + 10.0.13.1

![Capture P4-22](../assets/captures/P4/Capture_P4_22.png)

**<a id="p-07"></a> [P-07] · Propagation vers l'edge** — HQ-Router `O 172.16.2.0` + `O 172.16.3.0`

![Capture P4-19](../assets/captures/P4/Capture_P4_19.png)

**<a id="p-09"></a> [P-09] · SVI + durcissement** — DC-Leaf2 `Vlan220 up/up` + `interface status` (Gi1/0/5=220, reste 998)

![Capture P4-16](../assets/captures/P4/Capture_P4_16.png)

**<a id="p-10"></a> [P-10] · Est-Ouest (+ timeouts de boot)** — PC campus ping `.2.1` / `.2.12` / `.3.10` (3 timeouts→réponse, puis 4/4)

![Capture P4-15](../assets/captures/P4/Capture_P4_15.png)

**<a id="p-11"></a> [P-11] · Chemin Nord-Sud** — PC1 `tracert .2.11` via 10.0.3.10→10.0.12.1→10.0.9.10→10.0.4.2

![Capture P4-14](../assets/captures/P4/Capture_P4_14.png)

**<a id="p-12"></a> [P-12] · Compteur de sortie** — ASA `show nat`, `DC-NET translate_hits=3, untranslate_hits=2`

![Capture P4-03](../assets/captures/P4/Capture_P4_03.png)

**<a id="p-13"></a> [P-13] · Data-plane de sortie** — WEB-APP-1 `ping 8.8.8.8` réponse `TTL=249`

![Capture P4-04](../assets/captures/P4/Capture_P4_04.png)

**<a id="p-14"></a> [P-14] · Backend App1** — `http://172.16.2.11` = « Web-App 1 »

![Capture P4-11](../assets/captures/P4/Capture_P4_11.png)

**<a id="p-15"></a> [P-15] · Backend App2 (distinct)** — `http://172.16.2.12` = « Web-App 2 »

![Capture P4-10](../assets/captures/P4/Capture_P4_10.png)

**<a id="p-16"></a> [P-16] · LB (auto-documente la dette)** — `http://172.16.2.10` = page LB

![Capture P4-09](../assets/captures/P4/Capture_P4_09.png)

**<a id="p-17"></a> [P-17] · Isolation DMZ→APP** — ASA `DMZ-RESTRICT ligne 5 deny … hitcnt=4`

![Capture P4-05](../assets/captures/P4/Capture_P4_05.png)

**<a id="p-18"></a> [P-18] · Incident 3 — piège config-mode** — `DC-LEAF2(config)# show run interface` → `% Invalid input` ×3

![Capture P4-07](../assets/captures/P4/Capture_P4_07.png)

**<a id="p-19"></a> [P-19] · Incident 4 — ambre/PortFast** — ACC-SW1 `spanning-tree … fa0/3 detail` → « portfast mode », msg age 16

![Capture P4-06](../assets/captures/P4/Capture_P4_06.png)

**<a id="p-20"></a> [P-20] · Joignabilité edge vers le DC** — ASA `show route`, `3 subnets`, `S 172.16.2.0` + `S 172.16.3.0`

![Capture P4-32](../assets/captures/P4/Capture_P4_32.png)

**<a id="p-21"></a> [P-21] · Table NAT xlate (contexte)** — ASA `show xlate`, WEB-PUBLIC statique seul ; l'entrée dynamique `DC-NET` vieillit (sortie prouvée par le compteur [P-12])

![Capture P4-29](../assets/captures/P4/Capture_P4_29.png)

---

⬆️ [README de la partie](./README.md) · [Vue d'ensemble du projet](../README.md) — Suivant : [WORKFLOW Partie 5](../P5/WORKFLOW.md) (Téléphonie : CME, DHCP option 150, QoS).
