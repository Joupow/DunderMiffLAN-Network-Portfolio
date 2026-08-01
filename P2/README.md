# Partie 2 — Routage, redondance & services

**Bloc :** Siège TheBigOffice · **Outil :** Cisco Packet Tracer 9.0 · **Certification :** CompTIA Network+

> **Partie 2 · Routage, redondance & services**
> 
> Uplinks routés `/30` (`10.0.1–3.0`) · SVI réels `.2`/`.3`, VIP HSRP `.1` par VLAN · OSPF P2P aire 0 (RID `2.2.2.2`/`3.3.3.3`/`4.4.4.4`/`10.255.255.1`) · DHCP sur HQ-Router `10.0.1.2` · Core = transit L3 pur.
> 
> Plan d'adressage complet → [`IPAM.md`](../IPAM.md).

**Statut :** ✅ Validé — migration des SVIs Core→Distribution terminée, **redondance de passerelle prouvée** (failover HSRP + preempt, deux sens) · OSPF mono-aire point-à-point (4 adjacences `FULL`, zéro DR/BDR) · DHCP centralisé + relais mono-chemin · 2 incidents corrigés · 3 pièges de conception navigués · 0 déviation · 0 limitation PT

---
## Objectif

Migrer le routage inter-VLAN *temporaire* du Core (P1) sur la **Distribution en passerelles HSRP redondantes**, introduire des uplinks routés `/30` avec **OSPF** comme IGP du campus, et centraliser l'adressage hôte avec une **autorité DHCP unique + relais**. Cela clôt les deux dettes critiques de P1 (SVIs sur le Core, SPOF inter-VLAN).

**Contrainte structurante — la répartition.** Les deux VLANs lourds vivent sur des boîtiers différents, et pour chaque VLAN : **Active HSRP = root STP** (le *« service suit l'Active »*, hérité du root posé en P1).

| VLAN | Active HSRP = root STP | Standby |
|---|---|---|
| 10 (RH) · 30 (VOIP) | **DIST-SW1** | DIST-SW2 |
| 20 (IT) · 99 (MGMT) | **DIST-SW2** | DIST-SW1 |

La bascule est ordonnée **Core d'abord** : router ses uplinks et retirer ses SVIs data *avant* de lever les VIP, pour que `.1` ne soit jamais revendiqué par deux boîtiers. Trois pièges de conception ont été navigués (cible du helper, OSPF sur le HQ-Router, ordre de bascule) — détaillés en [WORKFLOW P2](./WORKFLOW.md).

![Topologie P2](../assets/topologies/topology_p2.svg)

---

## Niveaux & équipements

| Rôle | Équipement | Rôle dans la partie |
|---|---|---|
| Edge/Services | **HQ-Router** (ISR 2911) — *nouveau* | Serveur DHCP centralisé ; OSPF aire 0 ; futur edge ASA-inside (P3) |
| Core | Catalyst **3650** (L3) | **Transit L3 pur** — SVIs data retirés, `/30` routés, OSPF |
| Distribution | 2× Catalyst **3560** | **Passerelle L3** — SVIs + VIP HSRP par VLAN, uplink routé `/30`, OSPF, root STP |
| Access | 4× Catalyst **2960** (L2) | Inchangé — L2 dual-homed, VLANs trunkés |
| Postes | 8× PC | Migrés en **DHCP** (VLAN 10 & 20) ; passerelle = VIP HSRP |

Seul le lien HQ-Router est nouveau ; les uplinks Core↔DIST passent de trunk à routé `/30`. Le lien inter-Distribution (`Gi0/2`) **reste un trunk L2** — les hellos HSRP ont besoin du domaine de broadcast partagé. Câblage détaillé → [WORKFLOW P2, étape 1](./WORKFLOW.md).

---

## Couverture CompTIA Network+

| Domaine | Concept | Statut |
|---|---|---|
| Routage | OSPFv2 mono-aire (aire 0) | ✅ tous les voisins `FULL` |
| Routage | Type OSPF point-à-point | ✅ chaque `/30`, **zéro DR/BDR** |
| Routage | Router-ID (manuel) | ✅ codé en dur + `clear ip ospf process` |
| Routage | `passive-interface` sélectif | ✅ default + un-passive sur le transit uniquement |
| Routage | Ports routés (`no switchport`) | ✅ uplinks Core + DIST |
| Routage | ECMP | ✅ le Core atteint les VLANs via les deux DIST |
| Haute disponibilité | FHRP — HSRP (VIP / priorité / preempt) | ✅ failover **et** preempt prouvés, deux sens |
| Haute disponibilité | Répartition Active/Standby HSRP | ✅ DIST1 `{10,30}` · DIST2 `{20,99}` |
| Commutation | Root STP aligné sur l'Active HSRP | ✅ les quatre VLANs vérifiés |
| Services | Serveur DHCP (scopes, exclusions, options) | ✅ VLAN 10 & 20 sur HQ-Router |
| Services | Relais DHCP (`ip helper-address`) | ✅ chemin unique par VLAN |
| Adressage IP | Subnetting transit `/30` | ✅ 3 liens point-à-point, sans chevauchement |

---

## Matrice de validation (locale)

> Chaque `✅` cite un `[P-##]` de l'**[Annexe — Captures de preuve du WORKFLOW P2](./WORKFLOW.md#annexe--captures-de-preuve)**.

| ✅ Prouvé (par résultat, appel ou état) | ⚠️ Configuré / limité par PT |
|---|---|
| Uplinks routés du Core `up` (Gi1/0/1/2/24) — [P-01](./WORKFLOW.md#p-01) | Comportement du relais DHCP **pendant** un failover (dette #16 documentée, non capturée) |
| Adjacences OSPF `FULL/ -`, **zéro DR/BDR** — Core voit 3 voisins [P-02](./WORKFLOW.md#p-02) ; DIST1 [P-04](./WORKFLOW.md#p-04), DIST2 [P-06](./WORKFLOW.md#p-06), HQ [P-08](./WORKFLOW.md#p-08) | Joignabilité management complète depuis **chaque** ACC (vérifiée par sondage, non exhaustive) |
| Core `show ip route ospf` = ECMP vers les VLANs via les deux `/30` — [P-03](./WORKFLOW.md#p-03) ; routes DIST1 [P-05](./WORKFLOW.md#p-05) ; propagation HQ [P-07](./WORKFLOW.md#p-07) | |
| Répartition HSRP : DIST1 Active `{10,30}`, DIST2 Active `{20,99}`, preempt flaggé — [P-09](./WORKFLOW.md#p-09), [P-10](./WORKFLOW.md#p-10) | |
| Root STP = l'Active du VLAN : `{10}`→[P-11](./WORKFLOW.md#p-11), `{20}`→[P-12](./WORKFLOW.md#p-12), `{30}`→[P-13](./WORKFLOW.md#p-13), `{99}`→[P-14](./WORKFLOW.md#p-14) | |
| Baux DHCP distribués (`.10.50–.53`, `.20.51–.54`), **serveur unique** — [P-20](./WORKFLOW.md#p-20) | |
| Ping inter-VLAN PC V10 → `192.168.20.51` (TTL=127, un saut) — [P-21](./WORKFLOW.md#p-21) | |
| **Failover** : DIST1 SVI 10 coupé → DIST2 promu Active, ping rétabli (~3 perdus) — [P-15](./WORKFLOW.md#p-15), [P-16](./WORKFLOW.md#p-16), [P-17](./WORKFLOW.md#p-17) | |
| **Preempt** : DIST1 SVI 10 `no shutdown` → priorité 110 reprend l'Active — [P-18](./WORKFLOW.md#p-18), [P-19](./WORKFLOW.md#p-19) | |

> Le timeout sur le premier paquet d'un flux inter-VLAN frais (puis 0 %) est ARP + convergence, **pas** une faute.

---

## Registre d'erreurs & dette technique

> Registre complet en état final (points clos / portés / différés) et dépannage de session — **source unique : [WORKFLOW P2 §5](./WORKFLOW.md)**. Non dupliqué ici.

---

⬆️ **Suivant : [Partie 3 — DMZ & pare-feu](../P3/README.md)** — ASA 3 zones, NAT/PAT, 3 ACL, IDS/SPAN, origination de la route par défaut + verrou résumé/Null0. · [Vue d'ensemble du projet](../README.md)
