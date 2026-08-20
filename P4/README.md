# Part 4: Datacenter

 **Key concepts**: Spine-Leaf, Border Leafs, server tiers & load balancer

- 💻 **Tool**: Cisco Packet Tracer 9.0
- 🏷️ Full addressing plan → [IPAM](../IPAM.md)
- 📝 Step-by-step progression → [WORKFLOW P4](./WORKFLOW.md)
- 🎓 **Certification:** CompTIA Network+

## Logical topology

![P4 topology](../assets/topologies/topology_p4.svg)

## Objective

Build the datacenter as a **routed** Spine-Leaf fabric, wired into the campus **through the Core, not the HQ-Router**. The goal isn't "make the servers ping," it's to prove a precise traffic shape:

- uniform **East-West** (`Leaf→Spine→Leaf`)
- symmetric **North-South** (`Leaf→Spine→BorderLeaf→Core→edge`),
- and a **three-tier application** where the app servers egress but are never reachable inbound, each direction proven by a counter.

## Structural constraint

- Both Border Leafs terminate on the **Core** (`10.0.12.0/30`, `10.0.13.0/30`)
- The two N-S paths are identical in length → the Core learns the DC subnets via BL1 **and** BL2 in **ECMP**, and the HQ-Router stays a pure campus edge.

## Continuity decision (inherited from P3)

- The campus already reaches the Internet.
- The DC only needs OSPF to carry `172.16.2.0/24` + `172.16.3.0/24` to the edge, plus a dedicated PAT object. The ASA route + NAT is done **last**, once the fabric is proven.


## CompTIA Network+ coverage

| Domain | Concepts covered | Status                                                 |
|---|---|---|
| 🗺️ Topology & architecture | 2×2 spine-leaf fabric + 2 border leafs · East-West vs North-South traffic · 3-tier app (presentation/app/data) | ✅ E-W `Leaf→Spine→Leaf` · N-S via Border Leaf          |
| 🧭 Fabric routing | Routed access (no stretched VLAN) · point-to-point OSPF, no DR/BDR · equal-cost ECMP · folds into P3's `/20` summary | ✅ neighbors `FULL` · Core→DC via BL1 **and** BL2       |
| 🌐 Services | NAT/PAT for the new internal block | ✅ `DC-NET` PAT object                                  |
| 🛡️ Security | Tiered exposure (egress-only backend) · unused-port containment (VLAN 998) | ✅ egress ok / ingress denied · 998 across the fabric   |
| 🔁 High availability | Load balancer / VIP concept | ⚠️ documented: a working LB = production, out of scope for PT |

## Conclusion

A topology decision can buy you a property that no amount of config will ever recover: putting **both Border Leafs on the Core** makes the North-South paths symmetric (ECMP) and takes the HQ-Router out of the datacenter.

The deeper takeaway is thinking "by traffic direction": East-West uniform, North-South symmetric, an application tier that **egresses but never ingresses**.

Each direction has to be proven separately, because a flow that works one way tells you nothing about the other.

---

⬅️ [Part 3: DMZ & Firewall](../P3/README.md) · ⬆️ [Project overview](../README.md) · 🔁 [Workflow P4](./WORKFLOW.md) · **Next: [Part 5: VoIP](../P5/README.md)**: CME co-located with the VLAN 30 Active/root, DHCP Option 150, SCCP, TFTP, QoS boundary.
