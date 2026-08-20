# TheBigOffice · Network Portfolio
## 📓 Packet Tracer home lab for CompTIA Network+ (and beyond)

This repository presents **TheBigOffice**, a self-taught lab I built to put into practice the skills covered by **CompTIA Network+**, while exploring more advanced concepts. 

This network lab simulates an enterprise infrastructure under Cisco Packet Tracer in six parts, built progressively, each building on and extending the last.

| Part                 | Topic                      | **Key concepts**                                                                                |
| -------------------- | -------------------------- | ----------------------------------------------------------------------------------------------- |
| [P1](./P1/) | 🏗️ 3-tier LAN foundations | Cisco 3-tier hierarchical model · VLANs · 802.1Q · STP · inter-VLAN routing · SVI · Rapid PVST+ |
| [P2](./P2/) | 🧭 routing & redundancy    | Routing · HSRP · DHCP · Relay Helper · OSPF P2P                                                 |
| [P3](./P3/) | 🛡️ DMZ & firewall         | ASA · DMZ · NAT/PAT · filtering · ACL · IDS/SPAN                                                |
| [P4](./P4/) | 🖥️ Datacenter             | Spine-Leaf · Border Leafs · E-W and N-S traffic · ECMP · server tiers · load balancer · storage |
| [P5](./P5/) | 📞 VoIP                    | VoIP · CME · TFTP · DHCP Option 150 · voice QoS                                                 |
| [P6](./P6/) | 📶 WiFi                    | WLC · lightweight APs · CAPWAP · WPA2 Corp/Guest SSID · HSRPv2 VLAN 300                         |

The project is not intended to represent a turnkey production architecture. It is above all a **junior technical portfolio**; design mistakes are part of the learning process and are treated as opportunities for analysis and improvement.

## Project topology


![Global topology](./assets/topologies/topology-global.svg)

## The defining technical choices

- **learning a discipline of sequencing, not only the protocols configured.** The build order mattered to prevent failures _before_ they existed: STP root laid before redundancy (P1), Core routed and SVIs removed before raising the VIPs so as never to cause split-brain (P2), routing and NAT proven _without ACL_ before adding filtering (P3), datacenter fabric proven before wiring NAT (P4).

- **A single through-line: "the service follows the Active."** Per VLAN, HSRP Active = STP root = service hosted on the same box. A principle laid in P1 and inherited through to the CME (P5) and Wi-Fi (P6). Each part explicitly settles the debts of the previous one: the lab reads as **a single continuous engineering story**, not six isolated exercises.

- **Prove by a real state or traffic, never by a screen.** A blocked flow is proven at the ACL counter, a host by `REGISTERED in SCCP`, a DHCP authority by the fact that _no other server_ answers on the segment, never by a timeout or a capture that "looks like it works."

- **Debugging as the core of the learning.** The recurring pattern of the **return path** - the DHCP OFFER that doesn't know how to come back (P2), the default route that has to be injected into OSPF (P3), plus Router-ID collision and TFTP asymmetry: traps that are only understood by solving them.

- **Name the tool's limits, never hide them.** With the CAPWAP data plane not simulated, control (WLC, 4 APs `Online`) and data (autonomous AP) are proven separately. This is also what justifies closing at P6.

- **AI in supervision, not in authority**: AI was used as a tool for learning, review and design assistance, without ever replacing the understanding of networking concepts. It let me clarify Cisco syntax, identify inconsistencies and verify configurations throughout the build. 

## Documentation

The lab documentation is structured around global resources:

- 🏷️ [IPAM](./IPAM.md)
- 📘 [TECHNICAL_OVERVIEW](./TECHNICAL_OVERVIEW.md) 

Each part of the lab then has its own documentary space containing: 

- 📄 **README**
- 🪜 **WORKFLOW** 
- 🧪 **Cisco Packet Tracer file (.pkt)**

```
TheBigOffice - Packet Tracer Portfolio /
│
├── README.md               ← Showcase: project, skills, scope, repo map
├── TECHNICAL_OVERVIEW.md   ← Architecture, prod, decisions, learnings  
├── IPAM.md                 ← Addressing plan, VLAN/zones, Router-IDs, DHCP authority 
│
├── P1/ … P6/               ← One part per folder
│   ├── README.md           ← Frame: objective, scope, skills, validation matrix   
│   ├── WORKFLOW.md         ← reproduced: annotated CLI, step-by-step validation, 
│   └── TBO-Part_X.pkt/     ← lab .pkt files
│ 
├── assets/
│   ├── topologies/           ← topology_pX.svg + topology_global.svg
│   ├── network-overview/     ← Topology captures in Packet Tracer: N0_pX.png
│   └── captures/  P1/ … P6/  ← validation screenshots: Capture_PX_NN.png 
└──
```

This organization lets you follow the entire lifecycle of a network implementation:

- 🏛️ Network architecture design 
- 🧩 Segmentation and logical organization of the network
- ⚙️ Equipment configuration 
- ✅ Operation validation
- 🔎 Diagnosis and incident resolution
- 📈 Analysis of design limits and architecture improvement

## Initial plan

The initial plan included 11 parts with more topics to cover the majority of the CompTIA Network+ objectives, such as: IPv6, monitoring, PKI/RADIUS, cloud notions, attack simulation. 

The project stops at Part 6 because **Packet Tracer becomes the limiting factor**. 

Beyond a certain level, working around the simulator's limits (CAPWAP, VXLAN, iSCSI, SNAT…) costs more time than producing representative configuration.

An excellent learning tool, Packet Tracer remains a simulator: it models protocols without running a real IOS. 

**The continuation will perhaps be on GNS3 / Cisco CML**, or another virtualization software, which emulate real IOS images and make it possible to actually test what Packet Tracer can only represent, while continuing to build skills on professional tools.

## 🏁 Conclusion


The project went far beyond its original learning goal. 

Beyond CompTIA Network+ theory, this project demanded hands-on practice: designing a coherent architecture, implementing it and, above all, debugging it. 

Debugging produced the most real learning: routing loops, addressing conflicts, TFTP asymmetries and Router-ID collisions are not learned in a course; they are understood by solving them. 

The bet on learning by doing paid off, and it goes beyond mere preparation for the certification I've since earned. 
