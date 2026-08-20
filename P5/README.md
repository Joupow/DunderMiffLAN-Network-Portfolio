# Part 5: VoIP

 **Key concepts**: VoIP · CME · TFTP · DHCP Option 150 · voice QoS

- 💻 **Tool**: Cisco Packet Tracer 9.0
- 🏷️ Full addressing plan → [IPAM](../IPAM.md)
- 📝 Step-by-step progression → [WORKFLOW P5](./WORKFLOW.md)
- 🎓 **Certification:** CompTIA Network+

## Logical topology

![P5 topology](../assets/topologies/topology_p5.svg)

## Objective

Deploy IP telephony on the **VLAN 30 (voice)** with **Cisco CallManager Express**, covering a phone's full lifecycle:

- DHCP provisioning
- config over TFTP (Option 150)
- SCCP registration
- QoS prioritization

Every link **proven by a state command or a real call**, not by a screen.

The through-line is a **chain**: DHCP → TFTP → SCCP → registration; a single missing link breaks everything, and the symptom often surfaces three steps after the cause.
## Structural constraint

- CME lives on **DIST-SW1**, already the HSRP Active + STP root for VLAN 30. The service follows its Active.
- Since P4 took the Core fully L3, **VLAN 30 no longer exists on the Core**: voice is anchored on the Distribution layer, which makes this placement non-negotiable.
- Second pillar: **a single DHCP authority per domain**. HQ-Router for VLAN 10/20, **CME for VLAN 30**. The proof isn't that CME hands out leases, it's that **no other server** answers on VLAN 30.
## Continuity decision (inherited from P3)

- The ASA loop was already killed in P3 (`/20` summary + Null0).
- P5 **verifies**, it doesn't reconfigure.
## CompTIA Network+ coverage

| Domain | Concepts covered | Status |
|---|---|---|
| 🌐 Network services | inter-switch VoIP · DHCP option 150 (TFTP) · single DHCP authority per broadcast domain · pool segmentation (IPAM) | ✅ 1001↔1002 `Connected` · HQ (10/20) / CME (30) |
| 🔌 Switching | Voice VLAN + CDP advertisement · 802.1Q voice / untagged data on one cable | ✅ `Voice VLAN: 30` on ACC |
| 🎚️ QoS | DSCP EF (46) marking · conditional trust boundary | ✅ `trust device cisco-phone` |
| 📜 Protocols | SCCP (Skinny) TCP 2000 · TFTP · CDP | ✅ `REGISTERED in SCCP` |
| 🧭 Routing | Routes aligned with the allocated space | ✅ `/20` summary |
| 🛡️ Security | `ip tftp source-interface` (TFTP asymmetry) | 📋 documented: prod reference, not testable in PT |

## Conclusion

VoIP taught me to reason in dependency chains: DHCP → Option 150 → TFTP → SCCP is a chain where a single missing link breaks the whole phone.

Understanding that order is ~90% of VoIP troubleshooting.

Second takeaway, broader than voice: "a single DHCP authority" doesn't mean _one_ server for the whole network (that would be a SPOF), it means **one authority per broadcast domain**, and the proof isn't that the server hands out leases, it's that **no other one** answers on the segment.

---

⬅️ [Part 4: Datacenter](../P4/README.md) · ⬆️ [Project overview](../README.md) · 🔁 [Workflow P5](./WORKFLOW.md) · **Next: [Part 6: Wi-Fi](../P6/README.md)**: WLC + lightweight APs (CAPWAP), WPA2 Corp/Guest SSIDs, HSRPv2 VLAN 300, a deliberately hybrid architecture (data plane via a standalone AP).
