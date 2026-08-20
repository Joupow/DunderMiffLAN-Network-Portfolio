# Part 1: Workflow

**Key concepts**: Cisco 3-tier hierarchical model · VLANs · 802.1Q · STP · inter-VLAN routing · SVI 

- 💻 **Tool**: Cisco Packet Tracer 9.0
- 🏷️ Full addressing plan → [IPAM](../IPAM.md)
- 📄 Part 1 overview → [README P1](./README.md)
- 🎓 **Certification:** CompTIA Network+ 
## <a id="sommaire"></a>Contents

**1. Scope**

- [As-Built Topology](#Topologie-As-Built)
- [Tiers & equipment](#niveaux--équipements)

**2. Configuration steps**

- [Step 1: Physical deployment](#étape-1--déploiement-physique)
- [Step 2: Base VLANs](#étape-2--vlans-de-base)
- [Step 3: 802.1Q trunks](#étape-3--trunks-8021q)
- [Step 4: Access ports + hardening](#étape-4--ports-daccès--durcissement-de-bordure)
- [Step 4b: PC IP configuration](#étape-4b--config-ip-des-pc-gui)
- [Step 5: SVIs + inter-VLAN routing](#étape-5--svis--routage-inter-vlan-core)
- [Step 6: Management VLAN 99](#étape-6--vlan-de-management-99)
- [Step 7: STP – Rapid PVST+](#étape-7--stp--rapid-pvst-root-sur-la-distribution)
- [Step 8: Root on the Distribution](#étape-8-root-sur-la-distribution)

**3. Evidence & closure**

- [End-to-end validation](#validation-de-bout-en-bout-gate-final)
- [Troubleshooting (session incidents)](#dépannage-incidents-de-session)
- [Error log & technical debt](#registre-derreurs--dette-technique-état-final)
- [Appendix: Evidence captures](#annexe--captures-de-preuve)

# 1. Scope

## **<a id="Topologie-As-Built"></a>As-Built Topology**

PT diagram: L2 base, 3 tiers, dual-homed access, static addressing

![Networ-overview-P1](../assets/network-overview/NO_P1.png)

## <a id="niveaux--équipements"></a>Tiers & equipment

| Tier         | Equipment                 | Role in this part                                                |
| ------------ | ------------------------- | ---------------------------------------------------------------- |
| Core         | Catalyst **3650** (L3)    | Transport + temporary inter-VLAN routing (SVI gateways `.1`)     |
| Distribution | 2× Catalyst **3560**      | L2 aggregation, dual-homing, per-VLAN STP root, management       |
| Access       | 4× Catalyst **2960** (L2) | Connectivity for 8 PCs, dual-homed uplinks on `Fa0/1–2`          |
| Endpoints    | 8× PC                     | Static IP + per-VLAN gateway (transitional: DHCP in P2           |

# 2. Configuration steps

The order is not arbitrary: each step depends on the previous one. Creating a trunk before the VLANs exist locally, or an SVI before `ip routing`, produces silent failures diagnosed too late.

---
### <a id="étape-1--déploiement-physique"></a>Step 1: Physical deployment

**Intent:** build the topology and cabling in Packet Tracer. No CLI.

```
Core Gig1/0/1     <-> Gig0/1   DIST-SW1     (trunk, Gigabit)
Core Gig1/0/2     <-> Gig0/1   DIST-SW2     (trunk, Gigabit)
DIST-SW1 Gig0/2   <-> Gig0/2   DIST-SW2     (inter-Distribution: critical for HSRP in P2)
ACC-SW(x) Fa0/1   <-> Fa0/1-4  DIST-SW1     (primary uplink, 100M)
ACC-SW(x) Fa0/2   <-> Fa0/1-4  DIST-SW2     (redundant uplink, 100M)
ACC-SW(x) Fa0/3   <-> PC (odd)              (VLAN 10: RH)
ACC-SW(x) Fa0/4   <-> PC (even)             (VLAN 20: IT)
ACC-SW(x) Gig0/1-2: unused -> VLAN 998 + shutdown
```

> Access uplinks are on **ACC `Fa0/1` / `Fa0/2`**; the 2960's Gig ports are left unused (→ 998). The access trunk CLI (step 3) mirrors the ports actually cabled.

**Validation:** identify the ports that are physically up *before* configuring.

```cisco
show interfaces status
```

**Expected:** cabled ports (`Gig` Core/DIST uplinks, `Fa0/1-4` DIST, `Fa0/1-4` ACC) show `connected`; the rest `notconnect`. 

On screen, each link turns **amber** (STP listening/learning) then **green**; one uplink per ACC and one end of the inter-Distribution link stay amber = ports blocked by STP (standby), as expected.

> 📷 **[P-01](#p-01)** CORE `show interfaces status` (Gi1/0/1-2 trunk).

---
### <a id="étape-2--vlans-de-base"></a>Step 2: Base VLANs

**Intent:** declare every VLAN on every switch. A VLAN allowed on a trunk but missing from the local database carries **nothing**.

```cisco
enable
configure terminal

vlan 10
 name RH
vlan 20
 name IT
vlan 30
 name VOIP
vlan 99
 name MGMT
vlan 998
 name QUARANTINE
vlan 999
 name NATIVE_BLACKHOLE

end
write memory
```

**Validation:**

```cisco
show vlan brief
```

**Expected:** VLANs 10, 20, 30, 99, 998, 999 all `active`, on every switch.

> 📷 **[P-02](#p-02)** `show vlan brief`: canonical CORE (reproduced per switch for all 7).

---
### <a id="étape-3--trunks-8021q"></a>Step 3: 802.1Q trunks

**Intent:** bring up the inter-switch links with the black-hole native VLAN and VLAN 1 excluded. On PT 9.0, the `allowed vlan` list has to be retyped in full (no `add`).

> ⚠️ **PT / hardware limitation:** `switchport trunk encapsulation dot1q` is valid **only** on the 3560 (Distribution). It is rejected on the 3650 and the 2960, which handle 802.1Q natively.

**Core (3650): both uplinks:**

```cisco
enable
configure terminal

interface range gigabitEthernet 1/0/1 - 2
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,99,999
 switchport nonegotiate

end
write memory
```

**Distribution (3560): identical on DIST-SW1 and DIST-SW2:** 

```cisco
enable
configure terminal

! Uplink to the Core
interface gigabitEthernet 0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,99,999
 switchport nonegotiate

! Inter-Distribution link (Gigabit)
interface gigabitEthernet 0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,99,999
 switchport nonegotiate

! Downlinks to the access switches
interface range fastEthernet 0/1 - 4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,99,999
 switchport nonegotiate

end
write memory
```

> ⚠️ Skipping the `Fa0/1-4` block on the Distribution leaves those downlinks as **access VLAN 1**: they drop every tagged VLAN and CDP raises a `%CDP-4-NATIVE_VLAN_MISMATCH` facing the Access (native 999). This is exactly **incident #1** (§ Troubleshooting).

**Access (2960): both uplinks:**

```cisco
enable
configure terminal

interface range fastEthernet 0/1 - 2
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,99,999
 switchport nonegotiate

end
write memory
```

**Validation:**

```cisco
show interfaces trunk
```

**Expected:** Native vlan = 999 on every trunk; allowed lists complete and identical; DIST `Fa0/1-4` present as trunk.

> 📷 **[P-03](#p-03)** `show interfaces trunk`: DIST-SW1 + DIST-SW2, `Fa0/1-4` as trunk, native 999.

---
### <a id="étape-4--ports-daccès--durcissement-de-bordure"></a>Step 4: Access ports + edge hardening

**Intent:** assign the PC ports to their VLAN, enable the demonstrative Voice VLAN, isolate the unused ports, harden the ports facing endpoints.

> ⚠️ PortFast + BPDU Guard on host-facing ports ONLY (`Fa0/3`, `Fa0/4`), never on the `Fa0/1-2` uplinks.
> 
> BPDU Guard err-disables a port the moment it receives a BPDU; an uplink exchanges them continuously, so keeping it there kills the link on its first normal BPDU (**incident #2**). 
> 
> Loop protection on the redundant uplinks is STP's job (`Altn BLK`), not BPDU Guard's.

```cisco
enable
configure terminal

! Port to PC: VLAN 10 (RH)
interface fastEthernet 0/3
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable

! Port to PC: VLAN 20 (IT) + demonstrative Voice VLAN
interface fastEthernet 0/4
 switchport mode access
 switchport access vlan 20
 switchport voice vlan 30
 spanning-tree portfast
 spanning-tree bpduguard enable

! Unused ports -> quarantine + shutdown (including the unused Gig uplinks)
interface range fastEthernet 0/5 - 24
 switchport mode access
 switchport access vlan 998
 shutdown
interface range gigabitEthernet 0/1 - 2
 switchport mode access
 switchport access vlan 998
 shutdown

end
write memory
```

**Validation:**

```cisco
show vlan brief
show spanning-tree interface fastEthernet 0/3 portfast   ! -> enabled
```

> 📷 **[P-04](#p-04)** ACC `show interfaces status`: `Fa0/3`=10, `Fa0/4`=20, unused disabled/998.

---
### <a id="étape-4b--config-ip-des-pc-gui"></a>Step 4b: PC IP configuration (GUI)

**Intent:** give each PC a static IP, a `/24` mask and a default gateway consistent with its VLAN. 

**Transitional** build values (replaced by DHCP in P2): reference plan in [`IPAM.md`](../IPAM.md).

P1 allocation rule: **odd** PCs (1/3/5/7) → **VLAN 10**, IP `192.168.10.10–.13`; **even** PCs (2/4/6/8) → **VLAN 20**, IP `192.168.20.10–.13`; mask `255.255.255.0`; gateway = the VLAN's `.1` (Core SVI, transitional).

**Validation**: from a PC → Desktop → Command Prompt:

```cisco
ipconfig
```

---
### <a id="étape-5--svis--routage-inter-vlan-core"></a>Step 5: SVIs + inter-VLAN routing (Core)

**Intent:** enable the temporary inter-VLAN gateways on the Core.

> ⚠️ Dependency: `ip routing` **must** come before the `interface vlan` blocks. Without it, the 3650 stays L2 and the SVIs never route.
> 
> ⚠️ **Design debt:** these `.1` addresses are provisional; they migrate to the Distribution as HSRP VIPs in P2. **Never recreate a `.1` SVI on the Core after P2** (VIP conflict).

```cisco
enable
configure terminal

ip routing

interface vlan 10
 ip address 192.168.10.1 255.255.255.0
 no shutdown

interface vlan 20
 ip address 192.168.20.1 255.255.255.0
 no shutdown

interface vlan 30
 ip address 192.168.30.1 255.255.255.0
 no shutdown

interface vlan 99
 ip address 192.168.99.1 255.255.255.0
 no shutdown

end
write memory
```

**Validation:**

```cisco
show ip interface brief
show ip route            ! 4 connected 'C' routes: 10/20/30/99
```

> 📷 **[P-05](#p-05)** PC1 inter-VLAN ping `.20.10` + cross-switch intra-VLAN `.10.12`.

---
### <a id="étape-6--vlan-de-management-99"></a>Step 6: Management VLAN 99

**Intent:** make every switch reachable for administration from any VLAN.

> ⚠️ In P1 the 3560s are L3-capable but used as **pure L2** (inter-VLAN routing lives only on the Core). 
> 
> A pure-L2 switch consults no routing table: it needs `ip default-gateway` for its own management traffic to leave VLAN 99, just like the 2960s. 
> 
> Note: `ip default-gateway` is ignored if `ip routing` is active, so do **not** enable `ip routing` on the Distribution in P1.

**Distribution (3560s used as L2):**

```cisco
! DIST-SW1

enable
configure terminal

interface vlan 99
 ip address 192.168.99.11 255.255.255.0
 no shutdown
ip default-gateway 192.168.99.1

end
write memory
```

```cisco
! DIST-SW2

enable
configure terminal
interface vlan 99
 ip address 192.168.99.12 255.255.255.0
 no shutdown
ip default-gateway 192.168.99.1
end
write memory
```

**Access (2960: L2), adjust `.13/.14/.15/.16` per switch:**

```cisco
! ACC-SW1  (.14 / .15 / .16 for SW2 / SW3 / SW4)

enable
configure terminal

interface vlan 99
 ip address 192.168.99.13 255.255.255.0
 no shutdown
ip default-gateway 192.168.99.1

end
write memory
```

**Validation: two levels:**

```cisco
! From the Core: the SVIs answer within VLAN 99

ping 192.168.99.11
ping 192.168.99.12
ping 192.168.99.13
ping 192.168.99.14
ping 192.168.99.15
ping 192.168.99.16
```

Then from a VLAN 10 PC: `ping 192.168.99.1` – the management gateway must answer.

> 📷 **[P-06](#p-06)** DIST-SW2 `show ip interface brief` (Vlan99 `192.168.99.12` up/up) · **[P-07](#p-07)** PC1 ping `192.168.99.1`.

---
### <a id="étape-7--stp--rapid-pvst-root-sur-la-distribution"></a>Step 7: STP – Rapid PVST+

**Intent:** run **Rapid PVST+** (per-VLAN RSTP) across the whole L2 domain, then place the Root Bridge on the Distribution, aligned with the P2 Active split.

> ⚠️ **Set the mode on EVERY switch, not just the roots.** 
> 
> Rapid PVST+ negotiates its fast handshake (proposal/agreement) per link; a single switch left on classic `pvst` on that link drops the whole segment back to the slow 802.1D timers. 

**All switches: enable Rapid PVST+ (Core, DIST-SW1, DIST-SW2, each ACC-SW):**

```cisco
enable
configure terminal

spanning-tree mode rapid-pvst

end
write memory
```

---
### <a id="étape-8-root-sur-la-distribution"></a>Step 8: Root on the Distribution

> ⚠️ **The Core is NOT root, by design.** 
> 
> Do not run `root primary` on the Core. Setting `root primary/secondary` on the Distribution forces priorities (24576 / 28672) that beat the Core's default 32768.

**DIST-SW1: primary `{10,30,999}`, secondary `{20,99}`:**

```cisco
enable
configure terminal

spanning-tree vlan 10,30,999 root primary
spanning-tree vlan 20,99 root secondary

end
write memory
```

**DIST-SW2: primary `{20,99}`, secondary `{10,30,999}`:**

```cisco
enable
configure terminal

spanning-tree vlan 20,99 root primary
spanning-tree vlan 10,30,999 root secondary

end
write memory
```

> ⚠️ **Root scope.** Set deterministically on the Distribution for 10, 20, 30, 99 **and 999**. 
> 
> VLAN 999 carries no host (black-hole native), but its root is pinned on DIST-SW1 as a matter of hygiene. No access switch should be root of anything. 
> 
> **998** is not in the trunks' allowed list, no 998 BPDU crosses the links, each switch is the isolated root of its own local 998 instance.

**Validation:**

```cisco
show spanning-tree vlan 10     ! Root = DIST-SW1
show spanning-tree vlan 30     ! Root = DIST-SW1
show spanning-tree vlan 99     ! Root = DIST-SW2
show spanning-tree summary | include mode   ! rapid-pvst mode on every box
```

**Expected:** root matching the split; `Switch is in rapid-pvst mode` on all 7. The PortFast edge ports (`Fa0/3–4`) come up immediately; the redundant uplink stays `Altn BLK` but transitions via proposal/agreement instead of the 30 s listen/learn cycle.

> 📷 **[P-08](#p-08)** DIST-SW1 root `{10,30,999}` (rapid-pvst) · **[P-09](#p-09)** DIST-SW2 root `{20,99}` (rapid-pvst) · **[P-10](#p-10)** all 7 switches in `rapid-pvst mode`.

# 3. Evidence and closure

## <a id="validation-de-bout-en-bout-gate-final"></a>End-to-end validation 

| Domain          | Check            | Key command                                                | Expected                                                                            | Evidence                     |
| --------------- | ---------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------------------- | ---------------------------- |
| 🗺️ Topology    | Core uplinks     | `show interfaces status`                                   | Gi1/0/1-2 `connected trunk`                                                         | [P-01](#p-01)                |
| 🔌 Switching    | VLANs            | `show vlan brief`                                          | 10/20/30/99/998/999 active on all 7 switches                                        | [P-02](#p-02)                |
| 🔌 Switching    | Trunks           | `show interfaces trunk`                                    | DIST `Fa0/1-4` + Gig as trunk, native 999, full allowed list                        | [P-03](#p-03)                |
| 🔌 Switching    | Access edge      | `show interfaces status`                                   | `Fa0/3`=10, `Fa0/4`=20, unused disabled/998                                         | [P-04](#p-04)                |
| 🌳 STP          | STP root         | `show spanning-tree summary`                               | DIST1 root `{10,30,999}`, DIST2 root `{20,99}`                                      | [P-08](#p-08), [P-09](#p-09) |
| 🌳 STP          | STP mode         | `show spanning-tree summary`                               | `rapid-pvst mode` on all 7                                                          | [P-10](#p-10)                |
| 📦 Connectivity | Inter/intra-VLAN | PC1 `ping .20.10` + `.10.12`                               | reply (1st ARP packet tolerated)                                                    | [P-05](#p-05)                |
| 📦 Connectivity | Management       | DIST-SW2 `show ip int brief` + PC `ping .99.1`             | Vlan99 `.99.12` up/up; gateway answers                                              | [P-06](#p-06), [P-07](#p-07) |
| 🔁 High avail.  | **Failover**     | shut the `Root FWD` uplink of an ACC, re-ping the gateway  | blocked uplink (`Altn BLK`→`Root FWD`) promoted, traffic resumes; `no shutdown` restores | [P-11](#p-11)→[P-14](#p-14)  |

> ℹ️ A single `Request timed out` on the **first** packet of a new flow (then 0 %) is ARP resolution + STP convergence: not a fault. Persistent loss on an established flow is a real symptom.
>
> ⚠️ **Failover timing:** rapid-pvst confirmed ([P-10](#p-10)) → sub-second reconvergence expected; the **direct-measurement** ping under rapid-pvst is still to be captured.

## <a id="dépannage-incidents-de-session"></a>Troubleshooting (session incidents)

> Build history – incidents encountered and how each was caught. These are **not** debts: each was fixed within the same session..

| # | Symptom | Cause | Diagnosis | Fix |
|---|---|---|---|---|
| 1 | DIST downlink `connected 1` (not `trunk`); `%CDP-4-NATIVE_VLAN_MISMATCH` facing the Access | Distribution `Fa0/1-4` left as access VLAN 1 (step 3 downlink block not applied) | `show interfaces status` + `show interfaces trunk` | Apply the trunk config to DIST `Fa0/1-4` (native 999, allowed) · before: [P-TS1](#p-ts1) · after: [P-03](#p-03) |
| 2 | Uplink port red / GUI "On" box that unchecks itself | **BPDU Guard placed on an inter-switch uplink** → err-disable on the 1st legitimate BPDU | `show interfaces status err-disabled` (reason `bpduguard`) | `no spanning-tree bpduguard enable` + `no spanning-tree portfast` on the uplink; `shutdown`/`no shutdown` to recover; guard on `Fa0/3-4` only |
| 3 | 1st inter-VLAN ping: 1 timeout then replies (25 % then 0 %) | ARP resolution + STP convergence on the 1st packet | Re-ping → 0 % confirms | None: expected. `clear mac address-table dynamic` only if it persists |

**Reference commands:**

```cisco
show interfaces trunk        show ip route            show mac address-table
show vlan brief              show spanning-tree vlan  show interfaces status
show ip interface brief      show arp                 show running-config
show interfaces status err-disabled                   clear mac address-table dynamic
write memory
```

## <a id="registre-derreurs--dette-technique-état-final"></a>Error log & technical debt

> Final state of each item (closed / carried / deferred). Session troubleshooting is above. 
> ⚠️ **Cross-part numbering.** These numbers are identifiers referenced by other docs

| # | Item | Severity | Domain | Status |
|---|---|---|---|---|
| 1 | SVIs on the Core instead of the Distribution | 🟠 | L3 architecture | 🔜 P2 (HSRP): deliberate temporary simplification |
| 2 | A single Core = inter-VLAN routing SPOF | 🟠 | High availability | 🔜 P2 (HSRP on the Distribution) |
| 3 | STP root on the Distribution, split by HSRP Active (DIST1 `{10,30}`, DIST2 `{20,99}`) | 🟠 | STP | ✅ Done |
| 4 | Core + inter-Distribution links at Gigabit | 🟢 | Performance | ✅ Done |
| 4b | Access uplinks capped at 100M (the 3560 has only 2 Gig ports, both in use) | 🟢 | Performance | 📋 Hardware limit |
| 5 | STP root scope: trunked VLANs incl. 999; 998 isolated per switch | 🟠 | STP hardening | ✅ (998 per switch ⚠️) |
| 6 | Residual VLAN 1 on unused ports | 🟢 | Security | ✅ Done (998 + shutdown, Gig ports included) |
| 7 | PortFast + BPDU Guard on user ports | 🟠 | Edge security | ✅ Done (user ports only) |
| 8 | Port Security | 🟠 | Security | 🔜 P3 |
| 9 | Voice VLAN with no IP phone | 🟢 | Demonstrative | 🔜 P5 |
| 10 | Statically addressed PCs (no DHCP) | 🟢 | Scalability | 🔜 P2 (DHCP scopes + relay) |

## <a id="annexe--captures-de-preuve"></a>Appendix: Evidence captures

**<a id="p-01"></a> [P-01] · CORE `show interfaces status`**: Gi1/0/1-2 `connected trunk`

![Capture P1-10](../assets/captures/P1/Capture_P1_10.png)

**<a id="p-02"></a> [P-02] · VLAN database**: canonical CORE `show vlan brief`

![Capture P1-12](../assets/captures/P1/Capture_P1_12.png)

> Reproduced per switch: DIST-SW1 `Capture_P1_25` · DIST-SW2 `Capture_P1_24` · ACC-SW1 `Capture_P1_23` · ACC-SW2 `Capture_P1_22` · ACC-SW3 `Capture_P1_21` · ACC-SW4 `Capture_P1_20`.

**<a id="p-03"></a> [P-03] · Trunks (closes incident #1)**: DIST-SW1 + DIST-SW2 `show interfaces trunk`, `Fa0/1-4` as trunk, native 999

![Capture P1-13](../assets/captures/P1/Capture_P1_13.png)

![Capture P1-14](../assets/captures/P1/Capture_P1_14.png)

**<a id="p-04"></a> [P-04] · Access edge + hardening**: ACC-SW4 `show interfaces status` (`Fa0/3`=10, `Fa0/4`=20, unused disabled/998)

![Capture P1-11](../assets/captures/P1/Capture_P1_11.png)

> Reproduced on: ACC-SW3 `Capture_P1_17` · ACC-SW2 `Capture_P1_18` · ACC-SW1 `Capture_P1_19`.

**<a id="p-05"></a> [P-05] · Inter- & intra-VLAN**: PC1 `ping .20.10` (routed) + `.10.12` (cross-switch)

![Capture P1-07](../assets/captures/P1/Capture_P1_07.png)

**<a id="p-06"></a> [P-06] · Management SVI**: DIST-SW2 `show ip interface brief`, Vlan99 `192.168.99.12` up/up

![Capture P1-05](../assets/captures/P1/Capture_P1_05.png)

**<a id="p-07"></a> [P-07] · Management reachability**: PC1 `ping 192.168.99.1`

![Capture P1-06](../assets/captures/P1/Capture_P1_06.png)

**<a id="p-08"></a> [P-08] · STP root DIST-SW1 (rapid-pvst)**: `show spanning-tree summary`: `rapid-pvst mode`, Root for RH (10) + VOIP (30) + NATIVE_BLACKHOLE (999)

![Capture P1-17](../assets/captures/P1/Capture_P1_17.png)

**<a id="p-09"></a> [P-09] · STP root DIST-SW2 (rapid-pvst)**: `show spanning-tree summary`: `rapid-pvst mode`, Root for IT (20) + MGMT (99)

![Capture P1-15](../assets/captures/P1/Capture_P1_15.png)

**<a id="p-10"></a> [P-10] · Rapid PVST+ on every switch**: `show spanning-tree summary` = `Switch is in rapid-pvst mode`, canonical CORE

![Capture P1-16](../assets/captures/P1/Capture_P1_16.png)

> All-switch evidence (7/7): ACC-SW1 `Capture_P1_29` · ACC-SW2 `Capture_P1_37` · ACC-SW3 `Capture_P1_31` · ACC-SW4 `Capture_P1_32` · DIST-SW1 `Capture_P1_36` (root 10/30/999) · DIST-SW2 `Capture_P1_34` (root 20/99).

**<a id="p-11"></a> [P-11] · Failover: before**: ACC-SW1 STP v10: `Fa0/1` `Root FWD`, `Fa0/2` `Altn BLK`

![Capture P1-04](../assets/captures/P1/Capture_P1_04.png)

**<a id="p-12"></a> [P-12] · Failover: action**: ACC-SW1 `interface Fa0/1` → `shutdown`

![Capture P1-03](../assets/captures/P1/Capture_P1_03.png)

**<a id="p-13"></a> [P-13] · Failover: during**: PC1 `ping .10.1`: recovery after loss. ⚠️ Timing-measurement ping under rapid-pvst still to be re-taken.

![Capture P1-02](../assets/captures/P1/Capture_P1_02.png)

**<a id="p-14"></a> [P-14] · Failover: after**: ACC-SW1 STP v10: `Fa0/2` promoted to `Root FWD` (cost 23 via DIST-SW2)

![Capture P1-01](../assets/captures/P1/Capture_P1_01.png)

**<a id="p-ts1"></a> [P-TS1] · Incident #1 · Native mismatch (BEFORE fix)**: DIST-SW2 `%CDP-4-NATIVE_VLAN_MISMATCH` `Fa0/3-4
`
![Capture P1-09](../assets/captures/P1/Capture_P1_09.png)

> Stale companions (state *before* fix, troubleshooting only: never in validation): DIST-SW2 `Capture_P1_14` · DIST-SW1 `Capture_P1_15` (`Fa0/1-4 connected 1`).

---

⬆️ [Contents](#sommaire) · [Part README](./README.md) · [Project overview](../README.md) · **Next: [Workflow P2](../P2/WORKFLOW.md)** – migration of the SVIs to the Distribution, HSRP, OSPF P2P, DHCP + relay.
