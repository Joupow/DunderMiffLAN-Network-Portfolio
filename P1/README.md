# Part 1: Three-Tier LAN Foundations

**Key concepts**: Cisco three-tier hierarchical model · VLANs · 802.1Q · STP · inter-VLAN routing · SVI

- 💻 **Tool**: Cisco Packet Tracer 9.0  
- 🏷️ Full addressing plan → [IPAM](../IPAM.md)
- 📝 Step-by-step progression → [WORKFLOW P1](./WORKFLOW.md)
- 🎓 **Certification:** CompTIA Network+
## Logical topology

![P1 topology](../assets/topologies/topology_p1.svg)
## Objective

Architect the enterprise LAN foundation on Cisco's three-tier hierarchical model, comprising:

- Core, distribution, and access layers
- VLAN segmentation
- a dedicated management plane,
- STP stabilization
- *temporary* inter-VLAN routing on the Core.

Everything that follows: HSRP, OSPF, DMZ, datacenter, voice, Wi-Fi, hangs on how clean this base is.
## Structural constraint

- The STP root is placed on the **Distribution** layer from this part onward, aligned with the P2 HSRP plan (DIST-SW1 root `{10,30}`, DIST-SW2 root `{20,99}`).
- The L2 root and L3 default gateway therefore land on the same switch per VLAN starting in P2: the *"the service follows the Active"* principle, established here so the rest of the build inherits it.

**Why a temporary base on the Core?**

A redundant `.1` default gateway across two Distribution switches is **impossible without an FHRP** (the same IP on two chassis = a conflict). The deliberate call: ship a working single-chassis base first, then harden it with HSRP + a routed `/30` + OSPF in P2.

## CompTIA Network+ coverage

| Domain                 | Concepts covered                                          | Status                                               |
| ---------------------- | -------------------------------------------------------- | ---------------------------------------------------- |
| 🗺️ Topology           | Three-tier hierarchical · star / collapsed core (compared) | ✅ 7 switches, 3 tiers                                |
| 🔌 Switching           | VLANs · 802.1Q · hardened native VLAN · Rapid PVST+ · SVI | ✅ 6 VLANs across the 7 switches                      |
| 🔌 Switching           | Voice VLAN · speed/duplex                                | ⚠️ demonstrative (phone in P5) · access uplinks 100M |
| 🧭 Routing             | Inter-VLAN via the Core's SVIs                           | ✅ 4 SVIs · temporary → hardened in P2                |
| 🏷️ IPv4 addressing    | Class C `/24` · subnetting · static hosts               | ✅ 4× /24 · 8 hosts                                   |
| 🛡️ Security           | Port isolation · edge hardening                          | ✅ 998 + shutdown · BPDU Guard edge only              |
| 🔁 High availability   | L2 dual-homing · STP failover                            | ✅ mechanism proven · ⏳ sub-second timing to be measured |

## Lessons learned

A clean foundation isn't the "easy" step you rush through: it's the technical debt every later part inherits. This is pouring the foundation before the load-bearing walls go up: the L2 base has to be stable before you stack redundancy on top of it.

Two incidents caught this early: a native VLAN mismatch and BPDU Guard on an uplink, which would have contaminated HSRP, then OSPF, then voice had they slipped through.

The central takeaway remains the sequencing principle: place the STP root _before_ redundancy exists, aligned with the future HSRP Active, because a `.1` gateway made redundant across two switches is **impossible without an FHRP**.

---

⬆️ [Project overview](../README.md) · 🔁 [Workflow P1](./WORKFLOW.md) · **Next: [Part 2: Routing & Redundancy](../P2/README.md)**: migrating the SVIs onto the Distribution layer, HSRP, point-to-point OSPF, centralized DHCP + relay.
