# Part 6: Wi-Fi

 **Key concepts**: WLC · lightweight APs · CAPWAP · WPA2 Corp/Guest SSIDs · HSRPv2 VLAN 300

- 💻 **Tool**: Cisco Packet Tracer 9.0
- 🏷️ Full addressing plan → [IPAM](../IPAM.md)
- 📝 Step-by-step progression → [WORKFLOW P6](./WORKFLOW.md)
- 🎓 **Certification:** CompTIA Network+
## Logical topology

![P6 topology](../assets/topologies/topology_p6.svg)

## Objective

Lay an enterprise Wi-Fi layer on top of the P1–P5 wired foundation:

- a controller-based architecture (WLC + lightweight APs)
- SSID/VLAN segmentation,
- WPA2,
- and a gateway made redundant by **HSRPv2 on VLAN 300**, **proving every link with a state or real traffic**.

This is the part that introduces VLAN 300 to the network.

## Structural constraint

- Packet Tracer 9.0 **does not simulate the centralized CAPWAP data plane**.
- The WLC does register the APs (control plane proven), but it can't re-inject client traffic out its access port.
- Hence a **deliberately hybrid architecture**: the control plane is proven by the WLC (4 APs `Online`), the client data plane by a **standalone AP** (a direct bridge, with no controller in the path).

That's the only honest way to demonstrate both planes in the tool.


## Continuity decision (inherited from P5)

- DIST-SW1 is already the HSRP Active + STP root for VLANs 10/30
- VLAN 300 follows the same logic.
- **We never touch VLAN 30**: replaying a side-fix that flipped its root over to DIST-SW2 would break the CME/Active co-location from decision A1. Verified after the fact.


## CompTIA Network+ coverage

| Domain | Concepts covered | Status |
|---|---|---|
| 📶 Wireless: infra | AP/WLC · standalone vs lightweight · CAPWAP · AP groups | ✅ 4 LAP + standalone AP + WLC · 4 APs `Online` |
| 📶 Wireless: broadcast & security | SSID/BSSID/ESSID · WPA2-PSK (AES) · non-overlapping 2.4 GHz channel | ✅ Corp + Guest · channel 6 |
| 🔌 Switching | 802.1Q trunking · STP/PortFast/BPDU Guard · Wi-Fi VLAN segmentation | ✅ root VLAN 300 · hardened AP ports |
| 🧭 Routing | Wireless inter-VLAN | ✅ 300 → 10 via the standalone AP |
| 🔁 High availability | HSRPv2 Wi-Fi gateway | ✅ group 300, Active/Standby |
| 🌐 Services | DHCP for the APs | ✅ leases `.10-.13` |

**Assumed limits & debt (a tool constraint, not a design flaw)**

- 📋 WPA3 / 6 GHz / band steering · PSK-Enterprise (802.1X): unsupported / not simulable in Packet Tracer, documented as a prod reference.
- ⚠️ Guest network: configured, isolation not verifiable at the data plane under PT.
- ⚠️ VLANs 301/310: defined, not exercisable on the wire in PT (300 validated).
- 🔧 Channel 6 (2.4 GHz): assumed downgrade vs 5 GHz ch.36: **debt L14**.

## Conclusion

Packet Tracer doesn't simulate the centralized CAPWAP data plane; and the lesson isn't to quietly work around it, it's to **name the limitation honestly**:

control proven by the WLC (4 APs `Online`), data plane proven by a standalone AP. Recognizing what a tool _cannot_ demonstrate, and documenting it as such, is a sign of maturity, not an admission of weakness.

It's also what justifies closing the project here and moving to an emulator for what comes next.

---

⬅️ [Part 5: VoIP](../P5/README.md) · ⬆️ [Project overview](../README.md) · 🔁 [Workflow P6](./WORKFLOW.md) ·
