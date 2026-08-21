# Dunder MiffLan: IPAM
## Contents

- [IP addressing plan](#ip-addressing-plan)
- [1. VLAN plan / zones](#1-vlan-plan--zones)
- [2. Routed `/30` transit links: block `10.0.0.0/20`](#2-routed-30-transit-links-block-1000020)
- [3. OSPF Router-IDs (hardcoded)](#3-ospf-router-ids-hardcoded)
- [4. DHCP authority per domain](#4-dhcp-authority-per-domain)
- [5. Allocation conventions](#5-allocation-conventions)
- [6. Evolution by part](#6-evolution-by-part)

## IP addressing plan 

Three addressing spaces, separated by role:

- **Hosts & VLANs**: `192.168.0.0/16` (campus, voice, Wi-Fi) and `172.16.0.0/16` (DMZ, datacenter).

- **Internal routed transit**: contiguous block `10.0.0.0/20`, carved into point-to-point `/30`s. Single summary at the ASA, holes locked by Null0 (see P3).

- **Perimeter / edge**: public test ranges (RFC 5737) on the Internet side, plus the ASA↔HQ inside `/30`.

The plan is **stable from P1 to P6**: each part *adds* a segment, none re-addresses the existing one. The "introduced in" column of the last table traces this progression.

## 1. VLAN plan / zones

| Zone / VLAN                | Subnet                    | Role                                        | Gateway                            |
| -------------------------- | ------------------------- | ------------------------------------------- | ---------------------------------- |
| VLAN 10: HR                | `192.168.10.0/24`         | HR users                                    | `.1` (HSRP VIP: DIST-SW1 Active)   |
| VLAN 20: IT                | `192.168.20.0/24`         | IT users                                    | `.1` (HSRP VIP: DIST-SW2 Active)   |
| VLAN 30: VOIP              | `192.168.30.0/24`         | IP telephony                                | `.1` (HSRP VIP: DIST-SW1 Active)   |
| VLAN 99: MGMT              | `192.168.99.0/24`         | Equipment administration                    | `.1` (HSRP VIP: DIST-SW2 Active)   |
| VLAN 210: App DC           | `172.16.2.0/24`           | Application tier (APP-WEB1/2 + LB)          | `172.16.2.1` (DC-Leaf1 SVI)        |
| VLAN 220: Data DC          | `172.16.3.0/24`           | Data tier (SAN)                             | `172.16.3.1` (DC-Leaf2 SVI)        |
| VLAN 300: Wi-Fi mgmt       | `192.168.100.0/24`        | Wi-Fi management (WLC + LAP)                | `.1` (HSRPv2 VIP: DIST-SW1 Active) |
| VLAN 301: Corp             | (no IP on the wire in PT) | SSID `DunderWifLIN-Corp`                    | ⚠️ defined, not carried in PT      |
| VLAN 310: Guest            | (no IP on the wire in PT) | SSID `DunderWifLIN-Guest`                   | ⚠️ defined, not carried in PT      |
| DMZ zone                   | `172.16.0.0/24`           | Exposed servers (WEB-PUBLIC, PROXY)         | `172.16.0.1` (ASA dmz)             |
| VLAN 998: QUARANTINE       | -                         | Unused ports isolated + `shutdown`          | -                                  |
| VLAN 999: NATIVE_BLACKHOLE | -                         | Native VLAN of all trunks: untagged dropped | -                                  |

> `172.16.1.0/24` stays free between the DMZ (`.0`) and the DC (`.2`/`.3`): expansion reserve, unallocated.


## 2. Routed `/30` transit links: block `10.0.0.0/20`

**Campus (P2)**

| Link | Subnet | `.1` | `.2` |
|---|---|---|---|
| Core ↔ HQ-Router | `10.0.1.0/30` | Core | HQ-Router |
| Core ↔ DIST-SW1 | `10.0.2.0/30` | Core | DIST-SW1 |
| Core ↔ DIST-SW2 | `10.0.3.0/30` | Core | DIST-SW2 |

**Datacenter fabric (P4)**

| Link | Subnet | `.1` | `.2` |
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

> `10.0.0.0/30` (before `10.0.1.0`) and the space beyond `10.0.13.0/30` stay free in the `/20`: it is this empty space that the Null0 lock (AD 254) protects against loops (see P3).

**Perimeter / edge**

| Link | Subnet | `.1` / low | `.2` / high | Notes |
|---|---|---|---|---|
| Inside: ASA ↔ HQ-Router | `192.168.200.0/30` | ASA `.1` | HQ-Router `.2` | No OSPF on the ASA (statics) |
| Outside: ISP ↔ ASA | `203.0.113.0/30` | ISP `.1` | ASA `.2` | RFC 5737 · `.2` shared PAT + WEB-PUBLIC publication |
| External test: ISP ↔ PC | `198.51.100.0/24` | ISP `.1` | PC-EXTERIEUR `.10` | RFC 5737 · "Internet side" |

## 3. OSPF Router-IDs (hardcoded)

| Equipment | RID | Equipment | RID |
|---|---|---|---|
| CORE-SW | `10.255.255.1` | DC-Spine1 | `41.41.41.41` |
| DIST-SW1 | `2.2.2.2` | DC-Spine2 | `42.42.42.42` |
| DIST-SW2 | `3.3.3.3` | DC-Leaf1 | `43.43.43.43` |
| HQ-Router | `4.4.4.4` | DC-Leaf2 | `44.44.44.44` |
| | | DC-BorderLeaf1 | `45.45.45.45` |
| | | DC-BorderLeaf2 | `46.46.46.46` |

> **Deliberately absent:** the **ASA** does not participate in OSPF (static routes + `default-information originate` on the HQ side); the **CME** anchors VLAN 30 at L2 (direct broadcast, no routing); the **ISP-Router** simulates the Internet. None has a RID by design, not by omission.

---

## 4. DHCP authority per domain

> **Rule: a single DHCP authority per broadcast domain.** Not a single server for the whole network (SPOF), not two servers on the same segment. The proof is not merely that a server hands out leases: it is that **no other** one answers on the segment.

| VLAN           | Authority  | Distribution                                                                          |
| -------------- | ---------- | ------------------------------------------------------------------------------------- |
| 10: HR         | HQ-Router  | Relay via `ip helper-address` through the Active SVI (DIST-SW1)                       |
| 20: IT         | HQ-Router  | Relay via `ip helper-address` through the Active SVI (DIST-SW2)                       |
| 30: VOIP       | CME-Router | Direct broadcast, no helper: pool `VOIP_PHONES`                                       |
| 99: MGMT       | -          | Static (infrastructure)                                                               |
| 210 / 220: DC  | -          | Static (servers)                                                                      |
| 300: Wi-Fi     | DIST-SW1   | Direct broadcast: pool `VLAN300` · WLC internal DHCP **off** (anti double-server)     |

## 5. Allocation conventions

**Gateways & SVIs**

- `.1` = gateway of each L3 VLAN: shared **HSRP/HSRPv2 VIP** (10/20/30/99/300), direct IP for the DC leaf SVIs (`172.16.2.1`, `172.16.3.1`) and the DMZ (`172.16.0.1`, ASA).
- Real Distribution SVIs: `.2` = DIST-SW1, `.3` = DIST-SW2 (e.g. `192.168.30.2`/`.3`, `192.168.100.2`/`.3`).

**Service addresses (high or fixed)**

- CME-Router `192.168.30.254` · Generic WLC `192.168.100.200` · WLC 3504 (prod ref., disconnected) `192.168.100.201` · IDS-Sensor `192.168.99.20`.
- DMZ servers: WEB-PUBLIC `172.16.0.10`, PROXY `172.16.0.20`.
- DC servers: LB-APP (VIP) `172.16.2.10`, APP-WEB1 `172.16.2.11`, APP-WEB2 `172.16.2.12`, SAN `172.16.3.10`.

**Lease ranges & exclusions**

- VLAN 30: leases `192.168.30.50–.99` · exclusions `.1–.49` and `.100–.254`.
- VLAN 300: leases `192.168.100.10–.50` · exclusions `.1–.9` and `.51–.254`.
- VLAN 10 / 20: leases from `.50` · exclusions `.1–.49` (`ip dhcp excluded-address .1 .49` on the HQ-Router: see WORKFLOW P2 §5).
- Low addresses (`.1`–`.9`) reserved for gateways, VIPs and services; never handed out by DHCP.

**Structural VLANs**

- Native VLAN `999` (blackhole) on all trunks: untagged traffic dropped.
- VLAN `998` (quarantine): unused ports, in `shutdown`.

**Masks**

- Internal transit in `/30` (and not `/31`): two IPs "lost" per link, a deliberate choice for teaching-friendly readability.


## 6. Evolution by part

| Introduced in | Segment(s)                                                                                          | Detail                                                              |
| ------------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| [P1](./P1/)   | VLAN 10 / 20 / 30 / 99 / 998 / 999                                                                  | VLANs created; temporary `.1` gateways on Core SVIs                 |
| [P2](./P2/)   | Transit `10.0.1–3.0/30`                                                                             | OSPF area 0; `.1` gateways migrated to HSRP VIP on the Distribution |
| [P3](./P3/)   | DMZ `172.16.0.0/24` · outside `203.0.113.0/30` · inside `192.168.200.0/30` · test `198.51.100.0/24` | 3-zone ASA perimeter; Internet egress                               |
| [P4](./P4/)   | Fabric `10.0.4–13.0/30` · VLAN 210 `172.16.2.0/24` · VLAN 220 `172.16.3.0/24`                       | Routed Spine-Leaf datacenter; 2 Border Leafs on the Core            |
| [P5](./P5/)   | (VLAN 30 activated)                                                                                 | CME `192.168.30.254`; hosts in DHCP `.50–.99`                       |
| [P6](./P6/)   | VLAN 300 `192.168.100.0/24` · VLAN 301 · VLAN 310                                                   | Wi-Fi management + Corp/Guest SSID; HSRPv2 VLAN 300                 |
