# Partie 1 — Fondations LAN du siège

**Bloc :** Siège TheBigOffice · **Outil :** Cisco Packet Tracer 9.0 · **Certification :** CompTIA Network+

> **Partie 1 · LAN 3 niveaux**
> 
> VLANs `10` RH · `20` IT · `30` VOIP · `99` MGMT · `998` quarantaine · `999` natif trou noir · passerelles `.1` **temporaires sur SVI du Core** · mgmt VLAN 99 (`.99.11/.12` DIST, `.13–.16` ACC) · uplinks access `Fa0/1–2` · root STP sur la Distribution.
>
> Plan d'adressage complet → [`IPAM.md`](../IPAM.md).

**Statut :** ✅ Validé — fondation L2 prouvée de bout en bout, **failover inclus** · 2 incidents de build corrigés · 0 déviation · 0 limitation PT (1 contrainte matérielle : uplinks access 100M)

---

## Objectif

Construire la fondation du LAN d'entreprise sur le modèle hiérarchique Cisco à trois niveaux : segmentation VLAN, plan de management dédié, stabilisation STP, routage inter-VLAN *temporaire* sur le Core. Tout ce qui suit — HSRP, OSPF, DMZ, datacenter, voix, Wi-Fi — dépend de la propreté de cette base.

**Contrainte structurante.** Le root STP est positionné dès cette partie sur la **Distribution**, aligné sur le plan HSRP de P2 (DIST-SW1 root `{10,30}`, DIST-SW2 root `{20,99}`). Root L2 et passerelle L3 finiront ainsi sur le même switch par VLAN à partir de P2 — le principe *« le service suit l'Active »*, posé ici pour être hérité par toute la suite.

> **Pourquoi une base temporaire est de l'ingénierie légitime.** Une passerelle `.1` redondante sur deux switches de Distribution est **impossible sans FHRP** (même IP sur deux boîtiers = conflit). Livrer d'abord une base mono-boîtier qui fonctionne, puis la durcir avec HSRP + `/30` routé + OSPF en P2, est délibéré — pas une omission.

![Topologie P1](../assets/topologies/topology_p1.svg)

---

## Niveaux & équipements

| Rôle | Équipement | Rôle dans la partie |
|---|---|---|
| Core | Catalyst **3650** (L3) | Transport + routage inter-VLAN temporaire (passerelles SVI `.1`) |
| Distribution | 2× Catalyst **3560** | Agrégation L2, dual-homing, root STP par VLAN, management |
| Access | 4× Catalyst **2960** (L2) | Connectivité de 8 PC, uplinks dual-homed sur `Fa0/1–2` |
| Postes | 8× PC | IP statique + passerelle par VLAN (transitoire — DHCP en P2) |

Chaque switch d'accès est **dual-homed** (`Fa0/1` → DIST-SW1, `Fa0/2` → DIST-SW2) ; STP garde un uplink en forwarding et bloque l'autre par VLAN. Câblage détaillé → [WORKFLOW P1, étape 1](./WORKFLOW.md).

---

## Couverture CompTIA Network+

| Domaine | Concept | Statut |
|---|---|---|
| Topologies | Hiérarchique 3 niveaux | ✅ (simplifié — SVI sur le Core) |
| Topologies | Star / hub-and-spoke ; collapsed core | ✅ Access → Distribution ; comparé |
| Commutation | Base de données VLAN (création locale) | ✅ sur chaque switch |
| Commutation | Tag 802.1Q | ✅ tous les liens inter-switch |
| Commutation | Durcissement du VLAN natif | ✅ VLAN 999 des deux côtés |
| Commutation | Spanning Tree | ✅ **Rapid PVST+ sur les 7 switches** · root primary/secondary par VLAN, pré-aligné sur HSRP |
| Commutation | Voice VLAN | ⚠️ Démonstratif (poste en P5) |
| Commutation | SVI (data + management) | ✅ Core + management |
| Commutation | Speed / duplex | ⚠️ Gigabit core/inter-Dist ; uplinks access 100M (le 3560 n'a que 2 ports Gig) |
| Routage | Routage inter-VLAN | ✅ via SVI du Core (temporaire) |
| Adressage IP | IPv4 classe C, subnetting | ✅ 4× `/24` |
| Adressage IP | Adressage hôte statique | ✅ 8 PC, IP + passerelle par VLAN |
| Sécurité | Isolation de ports | ✅ VLAN 998 + shutdown |
| Sécurité | Durcissement de bordure | ✅ PortFast + BPDU Guard sur les ports utilisateur uniquement ; trunks `nonegotiate` |
| Haute disponibilité | Dual-homing L2 | ✅ **mécanisme de failover prouvé** · rapid-pvst confirmé (reconvergence sous-seconde attendue ; ping de timing direct à capturer) |

---

## Matrice de validation (locale)

> Chaque `✅` cite un `[P-##]` de l'**[Annexe — Captures de preuve du WORKFLOW P1](./WORKFLOW.md#annexe--captures-de-preuve)**.

| ✅ Prouvé (par résultat, appel ou état) | ⚠️ Configuré / limité par PT |
|---|---|
| Topologie as-built + uplinks Core `connected trunk` — [P-00](./WORKFLOW.md#p-00), [P-01](./WORKFLOW.md#p-01) | Joignabilité management de **chaque** ACC (seuls PC→`.99.1` et DIST-SW2 montrés individuellement) |
| VLANs présents sur tous les switches — [P-02](./WORKFLOW.md#p-02) | **Timing de failover sous-seconde** — mode rapid-pvst prouvé ([P-10](./WORKFLOW.md#p-10)) donc *attendu*, mais le ping de mesure directe reste à tirer |
| Trunks natif 999, listes allowed complètes — [P-03](./WORKFLOW.md#p-03) | |
| Bordure access : `Fa0/3`=10, `Fa0/4`=20, inutilisés 998/disabled — [P-04](./WORKFLOW.md#p-04) | |
| Root STP : DIST1 `{10,30,999}`, DIST2 `{20,99}` — [P-08](./WORKFLOW.md#p-08), [P-09](./WORKFLOW.md#p-09) | |
| **Mode Rapid PVST+ sur les 7 switches** — [P-10](./WORKFLOW.md#p-10) | |
| Ping inter-VLAN PC1(V10) → `192.168.20.10` + intra-VLAN cross-switch → `192.168.10.12` — [P-05](./WORKFLOW.md#p-05) | |
| Ping management → `192.168.99.1` · SVI 99 de DIST-SW2 up/up sur `.99.12` — [P-06](./WORKFLOW.md#p-06), [P-07](./WORKFLOW.md#p-07) | |
| **Mécanisme de failover** : ACC-SW1 `Fa0/1` (`Root FWD`) coupé → `Fa0/2` (`Altn BLK`) promu, ping rétabli — [P-11](./WORKFLOW.md#p-11)→[P-14](./WORKFLOW.md#p-14) | |

> Le timeout sur le tout premier paquet d'un ping inter-VLAN (25 % de perte, puis 0 %) est de la résolution ARP + convergence STP, **pas** une faute.

---

## Registre d'erreurs & dette technique

> Registre complet en état final (chaque point clos / porté / différé) et dépannage de session — **source unique : [WORKFLOW P1 §5](./WORKFLOW.md)**. Non dupliqué ici.

---

⬆️ **Suivant : [Partie 2 — Routage, redondance & services](../P2/README.md)** — migration des SVIs sur la Distribution, HSRP, OSPF point-à-point, DHCP centralisé + relais. · [Vue d'ensemble du projet](../README.md)
