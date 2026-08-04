# TheBigOffice : IPAM
## Sommaire

- [Plan d'adressage IP](#plan-dadressage-ip)
- [1. Plan VLAN / zones](#1-plan-vlan--zones)
- [2. Liens de transit routés /30 : bloc 10.0.0.0/20](#2-liens-de-transit-routés-30--bloc-1000020)
- [3. Router-IDs OSPF (codés en dur)](#3-router-ids-ospf-codés-en-dur)
- [4. Autorité DHCP par domaine](#4-autorité-dhcp-par-domaine)
- [5. Conventions d'allocation](#5-conventions-dallocation)
- [6. Évolution par partie](#6-évolution-par-partie)

## Plan d'adressage IP 

Trois espaces d'adressage, séparés par rôle :

- **Hôtes & VLAN** — `192.168.0.0/16` (campus, voix, Wi-Fi) et `172.16.0.0/16` (DMZ, datacenter).

- **Transit routé interne** — bloc contigu `10.0.0.0/20`, découpé en `/30` point-à-point. Résumé unique à l'ASA, trous verrouillés par Null0 (voir P3).

- **Périmètre / edge** — plages publiques de test (RFC 5737) côté Internet, plus le `/30` inside ASA↔HQ.

Le plan est **stable de P1 à P6** : chaque partie *ajoute* un segment, aucune ne réadresse l'existant. La colonne « introduit en » de la dernière table trace cette progression.

## 1. Plan VLAN / zones

| Zone / VLAN                 | Sous-réseau                   | Rôle                                          | Passerelle                          |
| --------------------------- | ----------------------------- | --------------------------------------------- | ----------------------------------- |
| VLAN 10 — RH                | `192.168.10.0/24`             | Utilisateurs RH                               | `.1` (VIP HSRP — DIST-SW1 Active)   |
| VLAN 20 — IT                | `192.168.20.0/24`             | Utilisateurs IT                               | `.1` (VIP HSRP — DIST-SW2 Active)   |
| VLAN 30 — VOIP              | `192.168.30.0/24`             | Téléphonie IP                                 | `.1` (VIP HSRP — DIST-SW1 Active)   |
| VLAN 99 — MGMT              | `192.168.99.0/24`             | Administration des équipements                | `.1` (VIP HSRP — DIST-SW2 Active)   |
| VLAN 210 — DC applicatif    | `172.16.2.0/24`               | Tier applicatif (APP-WEB1/2 + LB)             | `172.16.2.1` (SVI DC-Leaf1)         |
| VLAN 220 — DC data          | `172.16.3.0/24`               | Tier data (SAN)                               | `172.16.3.1` (SVI DC-Leaf2)         |
| VLAN 300 — Wi-Fi mgmt       | `192.168.100.0/24`            | Management Wi-Fi (WLC + LAP)                  | `.1` (VIP HSRPv2 — DIST-SW1 Active) |
| VLAN 301 — Corp             | — (pas d'IP sur le fil en PT) | SSID `TheBigOffice-Corp`                      | ⚠️ défini, non porté en PT          |
| VLAN 310 — Guest            | — (pas d'IP sur le fil en PT) | SSID `TheBigOffice-Guest`                     | ⚠️ défini, non porté en PT          |
| Zone DMZ                    | `172.16.0.0/24`               | Serveurs exposés (WEB-PUBLIC, PROXY)          | `172.16.0.1` (ASA dmz)              |
| VLAN 998 — QUARANTINE       | —                             | Ports inutilisés isolés + `shutdown`          | —                                   |
| VLAN 999 — NATIVE_BLACKHOLE | —                             | VLAN natif de tous les trunks — untagged jeté | —                                   |

> `172.16.1.0/24` reste libre entre la DMZ (`.0`) et le DC (`.2`/`.3`) — réserve d'expansion, non alloué.


## 2. Liens de transit routés `/30` — bloc `10.0.0.0/20`

**Campus (P2)**

| Lien | Sous-réseau | `.1` | `.2` |
|---|---|---|---|
| Core ↔ HQ-Router | `10.0.1.0/30` | Core | HQ-Router |
| Core ↔ DIST-SW1 | `10.0.2.0/30` | Core | DIST-SW1 |
| Core ↔ DIST-SW2 | `10.0.3.0/30` | Core | DIST-SW2 |

**Fabric datacenter (P4)**

| Lien | Sous-réseau | `.1` | `.2` |
|---|---|---|---|
| Spine1 ↔ Leaf1 | `10.0.4.0/30` | DC-Spine1 | DC-Leaf1 |
| Spine1 ↔ Leaf2 | `10.0.5.0/30` | DC-Spine1 | DC-Leaf2 |
| Spine2 ↔ Leaf1 | `10.0.6.0/30` | DC-Spine2 | DC-Leaf1 |
| Spine2 ↔ Leaf2 | `10.0.7.0/30` | DC-Spine2 | DC-Leaf2 |
| Spine1 ↔ BorderLeaf1 | `10.0.8.0/30` | DC-Spine1 | DC-BorderLeaf1 |
| Spine1 ↔ BorderLeaf2 | `10.0.9.0/30` | DC-Spine1 | DC-BorderLeaf2 |
| Spine2 ↔ BorderLeaf1 | `10.0.10.0/30` | DC-Spine2 | DC-BorderLeaf1 |
| Spine2 ↔ BorderLeaf2 | `10.0.11.0/30` | DC-Spine2 | DC-BorderLeaf2 |
| BorderLeaf1 ↔ Core | `10.0.12.0/30` | DC-BorderLeaf1 | Core |
| BorderLeaf2 ↔ Core | `10.0.13.0/30` | DC-BorderLeaf2 | Core |

> `10.0.0.0/30` (avant `10.0.1.0`) et l'espace au-delà de `10.0.13.0/30` restent libres dans le `/20` — c'est cet espace vide que le verrou Null0 (AD 254) protège contre les boucles (voir P3).

**Périmètre / edge**

| Lien | Sous-réseau | `.1` / bas | `.2` / haut | Notes |
|---|---|---|---|---|
| Inside — ASA ↔ HQ-Router | `192.168.200.0/30` | ASA `.1` | HQ-Router `.2` | Pas d'OSPF sur l'ASA (statiques) |
| Outside — ISP ↔ ASA | `203.0.113.0/30` | ISP `.1` | ASA `.2` | RFC 5737 · `.2` partagé PAT + publication WEB-PUBLIC |
| Test externe — ISP ↔ PC | `198.51.100.0/24` | ISP `.1` | PC-EXTERIEUR `.10` | RFC 5737 · « côté Internet » |

## 3. Router-IDs OSPF (codés en dur)

| Équipement | RID | Équipement | RID |
|---|---|---|---|
| CORE-SW | `10.255.255.1` | DC-Spine1 | `41.41.41.41` |
| DIST-SW1 | `2.2.2.2` | DC-Spine2 | `42.42.42.42` |
| DIST-SW2 | `3.3.3.3` | DC-Leaf1 | `43.43.43.43` |
| HQ-Router | `4.4.4.4` | DC-Leaf2 | `44.44.44.44` |
| | | DC-BorderLeaf1 | `45.45.45.45` |
| | | DC-BorderLeaf2 | `46.46.46.46` |

> **Absents volontaires :** l'**ASA** ne participe pas à OSPF (routes statiques + `default-information originate` côté HQ) ; le **CME** ancre le VLAN 30 en L2 (broadcast direct, aucun routage) ; l'**ISP-Router** simule Internet. Aucun n'a de RID par conception, pas par omission.

---

## 4. Autorité DHCP par domaine

> **Règle : une seule autorité DHCP par domaine de broadcast.** Pas un serveur unique pour tout le réseau (SPOF), pas deux serveurs sur un même segment. La preuve n'est pas seulement qu'un serveur distribue des baux — c'est qu'**aucun autre** ne répond sur le segment.

| VLAN | Autorité | Distribution |
|---|---|---|
| 10 — RH | HQ-Router | Relais `ip helper-address` via le SVI Active (DIST-SW1) |
| 20 — IT | HQ-Router | Relais `ip helper-address` via le SVI Active (DIST-SW2) |
| 30 — VOIP | CME-Router | Broadcast direct, aucun helper — pool `VOIP_PHONES` |
| 99 — MGMT | — | Statique (infrastructure) |
| 210 / 220 — DC | — | Statique (serveurs) |
| 300 — Wi-Fi | DIST-SW1 | Broadcast direct — pool `VLAN300` · DHCP interne du WLC **off** (anti double-serveur) |

## 5. Conventions d'allocation

**Passerelles & SVI**

- `.1` = passerelle de chaque VLAN L3 — **VIP HSRP/HSRPv2** partagée (10/20/30/99/300), IP directe pour les SVI de leaf DC (`172.16.2.1`, `172.16.3.1`) et la DMZ (`172.16.0.1`, ASA).
- SVI réels de la Distribution : `.2` = DIST-SW1, `.3` = DIST-SW2 (ex. `192.168.30.2`/`.3`, `192.168.100.2`/`.3`).

**Adresses de service (haut ou fixe)**

- CME-Router `192.168.30.254` · Generic WLC `192.168.100.200` · WLC 3504 (réf. prod, déconnecté) `192.168.100.201` · IDS-Sensor `192.168.99.20`.
- Serveurs DMZ : WEB-PUBLIC `172.16.0.10`, PROXY `172.16.0.20`.
- Serveurs DC : LB-APP (VIP) `172.16.2.10`, APP-WEB1 `172.16.2.11`, APP-WEB2 `172.16.2.12`, SAN `172.16.3.10`.

**Plages de baux & exclusions**

- VLAN 30 — baux `192.168.30.50–.99` · exclusions `.1–.49` et `.100–.254`.
- VLAN 300 — baux `192.168.100.10–.50` · exclusions `.1–.9` et `.51–.254`.
- VLAN 10 / 20 — baux à partir de `.50` · exclusions `.1–.49` (`ip dhcp excluded-address .1 .49` sur le HQ-Router — voir WORKFLOW P2 §5).
- Adresses basses (`.1`–`.9`) réservées passerelles, VIP et services ; jamais distribuées par DHCP.

**VLAN structurants**

- VLAN natif `999` (blackhole) sur tous les trunks — trafic untagged jeté.
- VLAN `998` (quarantaine) — ports inutilisés, en `shutdown`.

**Masques**

- Transit interne en `/30` (et non `/31`) : deux IP « perdues » par lien, choix de lisibilité pédagogique assumé.


## 6. Évolution par partie

| Introduit en         | Segment(s)                                                                                          | Détail                                                                 |
| -------------------- | --------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| [P1](./P1/README.md) | VLAN 10 / 20 / 30 / 99 / 998 / 999                                                                  | VLAN créés ; passerelles `.1` temporaires sur SVI du Core              |
| [P2](./P2/README.md) | Transit `10.0.1–3.0/30`                                                                             | OSPF aire 0 ; passerelles `.1` migrées en VIP HSRP sur la Distribution |
| [P3](./P3/README.md) | DMZ `172.16.0.0/24` · outside `203.0.113.0/30` · inside `192.168.200.0/30` · test `198.51.100.0/24` | Périmètre ASA 3 zones ; sortie Internet                                |
| [P4](./P4/README.md) | Fabric `10.0.4–13.0/30` · VLAN 210 `172.16.2.0/24` · VLAN 220 `172.16.3.0/24`                       | Datacenter Spine-Leaf routé ; 2 Border Leafs sur le Core               |
| [P5](./P5/README.md) | (VLAN 30 activé)                                                                                    | CME `192.168.30.254` ; postes en DHCP `.50–.99`                        |
| [P6](./P6/README.md) | VLAN 300 `192.168.100.0/24` · VLAN 301 · VLAN 310                                                   | Management Wi-Fi + SSID Corp/Guest ; HSRPv2 VLAN 300                   |
