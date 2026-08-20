# TheBigOffice: Technical overview

## Contents

- [Objective and intent](#objective-and-intent)
- [How to read this repository](#how-to-read-this-repository)
- [What a reviewer will challenge](#what-a-reviewer-will-challenge)
- [Technical summary by part](#technical-summary-by-part)
- [Defining design decisions](#defining-design-decisions)
- [Production gaps](#production-gaps)
- [Anti-copy-paste proof](#anti-copy-paste-proof)

## Objective and intent

The lab simulates a small enterprise network in six functional blocks, each building on a base validated by the previous one:

- **P1: HQ LAN**: switching foundations.
- **P2: Routing & redundancy**: HSRP, OSPF, DHCP.
- **P3: Perimeter**: ASA firewall, DMZ, NAT.
- **P4: Datacenter**: routed Spine-Leaf fabric.
- **P5: Voice**: CME telephony.
- **P6: Wi-Fi**: WLC controller and APs.

The objective is to demonstrate how **switching, routing, security, services, high availability and troubleshooting** interact in a single coherent environment. 

The described state is the **as-built** state: each part delivers a clean base validated before the next, and the defining decisions are justified, not merely applied.

## How to read this repository

Three levels of documentation, with **one canonical home per content type**. 

| Content type                                        | Canonical home                                                                      |
| --------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Decisions + the *why* of a part                     | `README_PX`: Objective, Defining constraint, Continuity decision, Conclusion     |
| CompTIA Network+ coverage + status ✅/⚠️/❌          | `README_PX`: CompTIA Network+ coverage                                              |
| Full addressing plan                                | [`IPAM.md`](./IPAM.md)                                                              |
| Framing (as-built topology, tiers & equipment)      | `WORKFLOW_PX` §1                                                                    |
| Step-by-step CLI of a part                          | `WORKFLOW_PX` §2: Configuration steps                                              |
| Validation proofs + troubleshooting                 | `WORKFLOW_PX` §3: End-to-end validation, Troubleshooting                            |
| Corrections register + debt + PT limits             | `WORKFLOW_PX` §3: Error register & technical debt                                   |
| Consolidated production gaps                        | **this overview**: [Production gaps](#production-gaps)                              |


## What a reviewer will challenge

**"Why Packet Tracer?"**

PT is used to practice and document Network+-level concepts in a coherent simulated topology. It demonstrates configuration logic but does not model advanced features. 

The project stops at Part 6 because beyond it, each topic would require GNS3 / Cisco CML or physical equipment to validate real behavior. This limit is **documented, not hidden**.

**"Is it production-ready?"**

No, and the gaps are tracked explicitly ([Production gaps](#production-gaps)). It is a junior portfolio lab; honesty about the limits is what earns credibility.

**"What proves it isn't copy-paste?"**

Three things a copy-paste does not produce: 

- The design reasoning is **justified** ([Defining design decisions](#defining-design-decisions)); 
- The **build incidents are documented** as learnings ([Anti-copy-paste proof](#anti-copy-paste-proof)); 
- The **validation matrices are honest**. Each part distinguishes ✅ proven / ⚠️ configured not proven / ❌ not simulable, and a blocked flow is proven by the **ACL counter**, never by a timeout.

## Technical summary by part

Portfolio index. The detail lives in the `README_PX` and `WORKFLOW_PX` of each row.

| #           | Block                | Main equipment                                  | Main concepts                                                      | Notable point                                                                                                 |
| ----------- | -------------------- | ----------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| [P1](./P1/) | 3-tier LAN           | Core 3650, 2× Distribution 3560, 4× Access 2960 | VLANs, trunks, Rapid PVST+, management VLAN, native 999 black hole | STP roots on the Distribution, aligned with the P2 HSRP plan; no Access is root                               |
| [P2](./P2/) | Routing & redundancy | Core, Distribution, HQ-Router                   | Distributed HSRP, OSPF `/30`, DHCP per domain + single relay       | DHCP relay pointing to the HQ-Router (`10.0.1.2`), not the Core; HSRP Active distributed per VLAN             |
| [P3](./P3/) | DMZ & Firewall       | ASA 5506-X, ISP router, DMZ services            | Security-levels, 3 opposed ACLs, NAT/PAT, IDS/SPAN, port-security  | Default route originated into OSPF; internal summaries locked by Null0 AD 254 (on the HQ-Router)              |
| [P4](./P4/) | Datacenter           | Spines / Leafs / Border Leafs: all 3650         | Routed Spine-Leaf, OSPF fabric, app + storage tiers, LB VIP        | Two Border Leafs on the Core (ECMP); fabric entered into the `/20` without renumbering                        |
| [P5](./P5/) | VoIP                 | CME, IP phones, access switches                 | DHCP Option 150, SCCP, TFTP, QoS boundary                          | CME on DIST-SW1 (Active + root of VLAN 30); P3 inheritance verified — `/20` summary audited safe, no reconfig |
| [P6](./P6/) | WiFi                 | WLC, lightweight APs, autonomous AP             | CAPWAP, SSID Corp 301 / Guest 310, WPA2, HSRPv2 VLAN 300           | VLAN 300 consolidated on DIST-SW1 (Active + root + DHCP + WLC); control and data plane proven separately      |

## Defining design decisions

Map of the choices that run through the lab. Each row is developed in the corresponding `README_PX`.

| Domain | Decision | Why it matters |
|---|---|---|
| 🧭 Routing | Hardcoded OSPF Router-IDs | Avoids unstable adjacency behavior |
| 🧭 Routing / 🔒 Security | OSPF kept out of the management VLAN | Reduces control-plane exposure |
| 🧭 Routing / 🏷️ IPAM | Transit planned as one contiguous `/20` | The datacenter fabric enters the plan without end-to-end renumbering |
| 🧭 Routing / 🔒 Security | ASA inside summaries (`/16` + `/20`) locked by Null0 rejects (AD 254) on the HQ-Router | Neutralizes the loop toward unallocated space without enumerating each `/30`; LPM always makes a real OSPF route win, Null0 only catches the holes (see [Cross-cutting threads](#cross-cutting-threads)) |
| 🔁 High availability / 🧭 Routing | HSRP gateways distributed across the Distribution | Removes the gateway SPOF and spreads the load (heavy VLANs on different switches) |
| 🔁 High availability / 🌐 Services | STP root + HSRP Active + critical service co-located per VLAN | Removes the hairpin on the inter-Distribution link; a critical service follows its Active |
| 🌐 Services | One DHCP authority per broadcast domain | Removes duplicated or ambiguous DHCP replies |
| 🖥️ Datacenter | Two Border Leafs on the Core (all-3650 fabric) | Separates N-S from E-W and provides ECMP; the 3650 brings 4 downlinks per Spine |
| 📶 Wi-Fi | Control plane / data plane separation at the WLC | Avoids claiming a validation PT cannot prove |

## Production gaps

| Gap                        | Current state (lab, as-built)                                                                                                     | Production direction                                                 |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| OSPF authentication        | Not implemented                                                                                                                   | Authentication of adjacencies on the routed links                    |
| Management plane           | Partial SSH/AAA; no dedicated management IP (managed in-band, Loopback0 deferred)                                                | SSHv2, AAA, management ACL, centralized logging, Loopback0          |
| DHCP redundancy            | One authority per domain; single-path relay (helper on the Active side only), pool on the Active, no new lease if the Active goes down | Split-scope / HA, mirror on the Standby                             |
| ASA routing                | Static inside summaries (`/16` + `/20`) + Null0 locks (AD 254) on the HQ-Router; audited safe (P5)                               | Dynamic routing (OSPF) on the firewall                              |
| Wi-Fi data plane           | CAPWAP client path not simulated (proven by autonomous AP)                                                                       | Real WLC, Cisco CML, GNS3 or physical lab                          |
| Guest isolation (VLAN 310) | Configured, not proven in data plane (no PT captive portal)                                                                      | Guest anchor WLC + captive portal                                   |
| Datacenter load balancer   | VIP presented, conceptual round-robin (no LB engine in PT)                                                                       | Functional LB (HAProxy / F5), health-checks                        |
| VoIP control               | Single CME/TFTP point                                                                                                            | Redundant call control + TFTP services (Loopback)                  |
| Datacenter gateway         | One SVI per Leaf (SPOF per tier)                                                                                                 | Anycast Gateway via VXLAN EVPN                                     |


## Anti-copy-paste proof

Build incidents documented as learnings. Detailed registers in the `WORKFLOW_PX` (§3); this table is the cross-cutting aggregate.

| Incident (part)                                      | Impact                                            | Diagnosis / retained design                                                                                                        |
| ---------------------------------------------------- | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Distribution downlinks left in access VLAN 1 (P1)    | `%CDP-4-NATIVE_VLAN_MISMATCH`, tagged VLANs dropped | Spotted via `show interfaces trunk`; native 999 trunk applied to `Fa0/1-4`                                                        |
| BPDU Guard placed on an inter-switch uplink (P1)     | Port err-disable at the 1st legitimate BPDU       | BPDU Guard reserved for host ports; loop protection on uplinks is STP's job (`Altn BLK`)                                           |
| ASA summaries toward unallocated space (P3)          | ASA ↔ HQ loop until TTL                            | One Null0 lock (AD 254) per summarized block on the HQ-Router; LPM makes a real OSPF route win, Null0 only catches the holes       |
| Possible duplicated DHCP replies (P5)                | Lease conflicts                                    | One DHCP authority per broadcast domain, proven by the **absence** of any second server on the segment                            |
| Non-simulable CAPWAP client data plane in PT (P6)    | Client forwarding unverifiable                    | Control and data planes proven **separately**: WLC (4 APs `Online`) + autonomous AP (real client, TTL 127)                        |

---

⬆️ [Contents](#contents)
