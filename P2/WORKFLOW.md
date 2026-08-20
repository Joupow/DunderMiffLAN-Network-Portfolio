# Part 2: Workflow 

 **Key concepts**: Routing, HSRP, DHCP, OSPF P2P
 
- 💻 **Tool**: Cisco Packet Tracer 9.0
- 🏷️ Full addressing plan → [IPAM](../IPAM.md)
- 📄 Part 2 overview → [README P2](./README.md)
- 🎓 **Certification:** CompTIA Network+ 
## <a id="sommaire"></a>Contents

**1. Scope**

- [As-Built Topology](#topologie-as-built)
- [Tiers & equipment](#niveaux--équipements)

**2. Configuration steps**

- [Step 1: HQ-Router placement & cabling](#étape-1--placement--câblage-du-hq-router)
- [Step 2: CORE: routed uplinks + SVI removal](#étape-2--core--uplinks-routés--retrait-des-svis)
- [Step 3: DISTRIBUTION: ip routing + SVIs + HSRP](#étape-3--distribution--ip-routing--uplink-routé--svis--hsrp)
- [Step 4: Point-to-point OSPF (area 0)](#étape-4--ospf-point-à-point-mono-aire-0)
- [Step 5: Centralized DHCP](#étape-5--dhcp-centralisé-sur-le-hq-router)
- [Step 6: DHCP relay: single path](#étape-6--relais-dhcp--chemin-unique)

**3. Evidence & closure**

- [End-to-end validation)](#validation-de-bout-en-bout-gate-final)
- [Troubleshooting (session incidents)](#dépannage-incidents-de-session)
- [Error log & technical debt](#registre-derreurs--dette-technique)
- [Appendix: Evidence captures](#annexe--captures-de-preuve)

# 1. Scope

## <a id="topologie-as-built"></a>As-Built Topology

PT diagram: shift to L3/HA – OSPF, per-VLAN HSRP, distributed STP root, DHCP

![Networ-overview-P2](../assets/network-overview/NO_P2.png)

## <a id="niveaux--équipements"></a>Tiers & equipment

| Role | Equipment | Role in this part |
|---|---|---|
| Edge/Services | **HQ-Router** (ISR 2911): *new* | Centralized DHCP server; OSPF area 0; future ASA-inside edge (P3) |
| Core | Catalyst **3650** (L3) | **Pure L3 transit**: data SVIs removed, routed `/30` toward DIST + HQ, OSPF |
| Distribution | 2× Catalyst **3560** | **L3 gateway**: SVIs + per-VLAN HSRP VIP, routed `/30` uplink, OSPF, STP root |
| Access | 4× Catalyst **2960** (L2) | Unchanged: dual-homed L2, VLANs trunked toward the Distribution |
| Endpoints | 8× PC | Migrated to **DHCP** (VLAN 10 & 20); gateway = HSRP VIP |

Nothing is physically re-cabled **except** the new HQ-Router link (`HQ Gi0/0` ↔ `Core Gi1/0/24`). 

What changes is the **role** of the existing links: the Core↔DIST uplinks go from trunk to routed `/30`; the inter-Distribution link (`Gi0/2`) **stays an L2 trunk** (HSRP hellos need the shared broadcast domain).

# 2. Configuration steps

This is a **migration** from a live P1, not a greenfield. 

Raising the VIPs while the Core still holds `.1` on a connected trunk causes a duplicate-IP conflict. 

Routing the Core uplinks first brings its data SVIs down to `down/down` on their own, which lets the Distribution take `.1` cleanly.

---
### <a id="étape-1--placement--câblage-du-hq-router"></a>Step 1: HQ-Router placement & cabling

**Intent:** introduce the router that carries DHCP (and, in P3, the ASA-inside link). No CLI.

**Cable** (straight-through): **HQ `Gi0/0`** ↔ **Core `Gi1/0/24`**.

> ⚠️ Router interfaces are `administratively down` by default; the link will not come up until `no shutdown` is done on HQ `Gi0/0` (**incident #1**).

**Validation** (on the Core): `show interfaces status | include 1/0/24` → `notconnect` until the router interface is brought up in step 3.

---
### <a id="étape-2--core--uplinks-routés--retrait-des-svis"></a>Step 2: CORE: routed uplinks + SVI removal

**Intent:** turn the Core into pure L3 transit and strip it of its inter-VLAN gateway role. `default interface` wipes the residual P1 trunk config before the L3 config lands.

```cisco
enable
configure terminal

default interface GigabitEthernet1/0/1
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.0.2.1 255.255.255.252
 ip ospf network point-to-point
 no shutdown

default interface GigabitEthernet1/0/2
interface GigabitEthernet1/0/2
 no switchport
 ip address 10.0.3.1 255.255.255.252
 ip ospf network point-to-point
 no shutdown

interface GigabitEthernet1/0/24
 no switchport
 ip address 10.0.1.1 255.255.255.252
 ip ospf network point-to-point
 no shutdown

! Remove the temporary data + mgmt SVIs (frees .1 for the VIP)
no interface vlan 10
no interface vlan 20
no interface vlan 30
no interface vlan 99

end
write memory
```

> ⚠️ **Design pitfall #3: Core first.** Removing the Core's data SVIs *before* raising the VIPs avoids the window where the Core's `.1` SVI and the DIST's `.1` HSRP VIP are alive together = duplicate-IP conflict / gratuitous-ARP war.
> 
> ⚠️ Never recreate a `.1` SVI on the Core afterwards. `ip routing` stays enabled; it is a transit router, managed in-band via its `/30`s + OSPF.

**Validation:** `show ip interface brief | include 10.0.` → `Gi1/0/1=10.0.2.1`, `Gi1/0/2=10.0.3.1`, `Gi1/0/24=10.0.1.1`, all `up/up`. The data/mgmt SVIs no longer appear.

> 📷 **[P-01](#p-01)** CORE `show ip interface brief`: routed uplinks up.

---
### <a id="étape-3--distribution--ip-routing--uplink-routé--svis--hsrp"></a>Step 3: DISTRIBUTION: ip routing + routed uplink + SVIs + HSRP

**Intent:** the Distribution becomes the redundant gateway. `ip routing` first, otherwise the SVIs don't route. `Gi0/1` (Core uplink) → routed; **`Gi0/2` (inter-Distribution) stays an L2 trunk.**

**DIST-SW1: Active `{10,30}`, Standby `{20,99}`:**

```cisco
enable
configure terminal

ip routing

default interface GigabitEthernet0/1
interface GigabitEthernet0/1
 no switchport
 ip address 10.0.2.2 255.255.255.252
 ip ospf network point-to-point
 no shutdown

interface vlan 10
 ip address 192.168.10.2 255.255.255.0
 standby 10 ip 192.168.10.1
 standby 10 priority 110
 standby 10 preempt
 no shutdown
 
interface vlan 20
 ip address 192.168.20.2 255.255.255.0
 standby 20 ip 192.168.20.1
 standby 20 priority 100
 no shutdown
 
interface vlan 30
 ip address 192.168.30.2 255.255.255.0
 standby 30 ip 192.168.30.1
 standby 30 priority 110
 standby 30 preempt
 no shutdown
 
interface vlan 99
 ip address 192.168.99.2 255.255.255.0
 standby 99 ip 192.168.99.1
 standby 99 priority 100
 no shutdown
 
end
write memory
```

**DIST-SW2: exact mirror:** Active `{20,99}`, Standby `{10,30}`: real IPs `.3`, priority `110` + `preempt` on 20 & 99, `100` on 10 & 30, uplink `10.0.3.2/30`.

**Validation:** `show standby brief`
- DIST1: `Active` on Vl10/Vl30, `Standby` on Vl20/Vl99, `P` (preempt) present.
- DIST2: the exact mirror. `Virtual IP` column = each VLAN's `.1`.

> ⚠️ If a VLAN shows `Active` on **both** switches → the inter-Dist `Gi0/2` trunk isn't carrying that VLAN (split-brain). Check that its allowed list includes `10,20,30,99,999` (retype the full list: `add` rejected on PT 9.0).
> ⚠️ On an L3 switch (`ip routing` active), `ip default-gateway` is ignored: don't set it on the DIST; they learn their routes via OSPF.

> 📷 **[P-09](#p-09)** HSRP DIST-SW1 (Active V10/V30, Pri 110 P) · **[P-10](#p-10)** HSRP DIST-SW2 mirror (Active V20/V99).

---
### <a id="étape-4--ospf-point-à-point-mono-aire-0"></a>Step 4: Point-to-point OSPF (single area 0)

**Intent:** one area, all links point-to-point (no DR/BDR election), hard-coded RID, SVIs advertised but passive.

**CORE (transit only, no SVI to advertise):**

```cisco
configure terminal

router ospf 1
 router-id 10.255.255.1
 network 10.0.1.0 0.0.0.3 area 0
 network 10.0.2.0 0.0.0.3 area 0
 network 10.0.3.0 0.0.0.3 area 0
 
end
clear ip ospf process        ! answer "yes": required for the RID to take effect
```

**DIST-SW1 (RID `2.2.2.2`) / DIST-SW2 (RID `3.3.3.3`):**

```cisco
configure terminal

router ospf 1
 router-id 2.2.2.2
 passive-interface default
 no passive-interface GigabitEthernet0/1     ! ONLY the /30 transit forms an adjacency
 network 10.0.2.0 0.0.0.3 area 0             ! DIST2: 10.0.3.0 0.0.0.3
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
 network 192.168.99.0 0.0.0.255 area 0
 
end
clear ip ospf process
```

> `passive-interface default` = the VLAN subnets are **advertised** but never form an adjacency on an SVI (kills stray OSPF on VLAN 99). Since the inter-Dist `Gi0/2` is L2, DIST1 and DIST2 have **no** direct OSPF adjacency; they learn each other's prefixes via the Core.

**HQ-Router (RID `4.4.4.4`):**

```cisco
enable
configure terminal

interface GigabitEthernet0/0
 ip address 10.0.1.2 255.255.255.252
 ip ospf network point-to-point
 no shutdown
 
router ospf 1
 router-id 4.4.4.4
 network 10.0.1.0 0.0.0.3 area 0
 
end
clear ip ospf process
```

> ⚠️ **Design pitfall #2: the HQ-Router must run OSPF.** 
> 
> Without a return route to the VLAN subnets, the DHCP OFFER is generated but never routed back: silent failure. Same class of "return-path" pitfall as the default route in P3.

**Validation:**

```cisco
show ip ospf neighbor        ! all FULL/ -, no DR/BDR
show ip ospf interface brief ! every interface = P2P
show ip route ospf           ! Core: ECMP to the VLANs via both /30s; HQ: all VLANs present
```

**Expected:** the Core sees `4.4.4.4` (Gi1/0/24), `2.2.2.2` (Gi1/0/1), `3.3.3.3` (Gi1/0/2), all `FULL/ -`. HQ `show ip route ospf` lists `10.0.2/3` and `192.168.10/20/30/99`.

> 📷 **[P-02](#p-02)** Core OSPF neighbors (3 `FULL` neighbors) · **[P-03](#p-03)** Core ECMP to VLANs · **[P-04](#p-04)/[P-05](#p-05)** DIST-SW1 adjacency + routes · **[P-06](#p-06)** DIST-SW2 · **[P-07](#p-07)/[P-08](#p-08)** HQ-Router propagation + adjacency.

---
### <a id="étape-5--dhcp-centralisé-sur-le-hq-router"></a>Step 5: Centralized DHCP on the HQ-Router

**Intent:** one DHCP authority for the user VLANs. VLAN 30 = CME (P5); VLAN 99 = static.

**HQ-Router:**

```cisco
configure terminal

ip dhcp excluded-address 192.168.10.1 192.168.10.49
ip dhcp excluded-address 192.168.20.1 192.168.20.49

ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1        ! = HSRP VIP (survives a failover)
 dns-server 192.168.99.1
exit

ip dhcp pool VLAN20
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 192.168.99.1
 
end
write memory
```

---
### <a id="étape-6--relais-dhcp--chemin-unique"></a>Step 6: DHCP relay: single path

**Intent:** the helper lives on the SVI of the VLAN's **Active**, and on it alone.

> **Helper target:** the DHCP server is **HQ-Router = `10.0.1.2`**, not the Core `10.0.1.1`.

```cisco
! DIST-SW1: Active for VLAN 10

interface vlan 10
 ip helper-address 10.0.1.2

! DIST-SW2: Active for VLAN 20

interface vlan 20
 ip helper-address 10.0.1.2
```

> ⚠️ **Design pitfall #1: helper on the Active only, not both DIST.** 
> 
> VLAN 10 spans DIST1 and DIST2 (inter-Dist trunk). A DISCOVER is heard by both SVI 10s; two helpers → two relays → two OFFERs. The helper on the Active guarantees a single path.
> 
> 📋 **Debt #16 (accepted):** after a DIST1→DIST2 failover, DIST2's SVI 10 has no helper → no *new* lease in VLAN 10 while DIST1 is down (existing leases hold). Deliberate anti-double-relay trade-off.

**Validation:** switch a VLAN 10 PC to DHCP → `show ip dhcp binding` on the HQ-Router. Confirmed this build: leases `.10.50–.53` and `.20.51–.54`, single server.

> 📷 **[P-20](#p-20)** centralized DHCP leases (single server).

# 3. Evidence & closure

## <a id="validation-de-bout-en-bout-gate-final"></a>End-to-end validation 

| Domain | Check | Key command | Expected | Evidence |
|---|---|---|---|---|
| 🔗 L3 transit | Routed `/30` uplinks | Core `show ip interface brief` | Gi1/0/1=10.0.2.1 · Gi1/0/2=10.0.3.1 · Gi1/0/24=10.0.1.1, all up | [P-01](#p-01) |
| 🌐 OSPF | Core adjacencies | Core `show ip ospf neighbor` | 3× `FULL/ -` (2.2.2.2, 3.3.3.3, 4.4.4.4), no DR/BDR | [P-02](#p-02) |
| 🌐 OSPF | ECMP Core → VLANs | Core `show ip route ospf` | ECMP to all VLANs via 10.0.2.2 **and** 10.0.3.2 | [P-03](#p-03) |
| 🌐 OSPF | DIST1 adjacency + routes | DIST1 `show ip ospf neighbor` + `route ospf` | 10.255.255.1 via 10.0.2.1 | [P-04](#p-04), [P-05](#p-05) |
| 🌐 OSPF | DIST2 adjacency + routes | DIST2 `show ip ospf neighbor` + `route ospf` | 10.255.255.1 via 10.0.3.1 | [P-06](#p-06) |
| 🌐 OSPF | HQ-Router propagation | HQ `show ip route ospf` + `neighbor` | transits + VLANs via 10.0.1.1 | [P-07](#p-07), [P-08](#p-08) |
| 🔁 High avail. | HSRP split | `show standby brief` (both DIST) | DIST1 Active 10/30 · DIST2 Active 20/99 | [P-09](#p-09), [P-10](#p-10) |
| 🌳 STP | Root aligned to the Active | `show spanning-tree vlan 10/20/30/99` | root = the VLAN's Active (4× `This bridge is the root`) | [P-11](#p-11)→[P-14](#p-14) |
| 📦 Connectivity | Routed inter-VLAN | PC V10 `ping <PC V20>` | routed, TTL=127 (1st ARP packet then OK) | [P-21](#p-21) |
| 📡 Services | DHCP leases | HQ `show ip dhcp binding` | single server, leases .10.50–.53 / .20.51–.54 | [P-20](#p-20) |
| 🔁 High avail. | **Failover** | DIST1 `int vlan 10 / shutdown` + ping VIP `-t` | DIST2 promoted to Active, ping resumes (~3 lost) | [P-15](#p-15)→[P-17](#p-17) |
| 🔁 High avail. | **Preempt** | DIST1 `int vlan 10 / no shutdown` | priority 110 reclaims Active on DIST1 | [P-18](#p-18), [P-19](#p-19) |

## <a id="dépannage-incidents-de-session"></a>Troubleshooting (session incidents)

> Incidents encountered during the build, with the diagnostic that caught each one. Session history, **not** debts; each fixed the same day.

| # | Symptom | Cause | Diagnosis | Fix |
|---|---|---|---|---|
| 1 | Core `Gi1/0/24` stays `Down` despite `10.0.1.1/30` + `no switchport` | HQ-Router `Gi0/0` still `administratively down` (routers boot with interfaces shut) | `show ip interface brief` on HQ | `interface Gi0/0 / no shutdown` (+ IP + `ip ospf network p2p`) |
| 2 | DIST-SW2 `show ip ospf neighbor` empty; the Core never sees `3.3.3.3` | `network 10.0.3.0 0.0.0.3 area 0` missing: OSPF never enabled on the transit | `show ip ospf neighbor` (DIST2 vs DIST1) + cross-check on the Core | Add the `network` line + `clear ip ospf process` |

**Reference commands:**

```cisco
show ip ospf neighbor        show ip ospf interface brief    show ip route ospf
show standby brief           show spanning-tree vlan         show ip dhcp binding
show ip interface brief      show interfaces trunk           show running-config
clear ip ospf process        write memory
```

## <a id="registre-derreurs--dette-technique"></a>Error log & technical debt

> Final state of each item (closed / carried / deferred). Session troubleshooting is above. 
> ⚠️ **Cross-part numbering.** These numbers are identifiers referenced by other docs

| #   | Item                                              | Severity | Domain              | Status                                                                                  |
| --- | ------------------------------------------------- | -------- | ------------------- | --------------------------------------------------------------------------------------- |
| 1   | SVIs on the Core instead of the Distribution      | 🟠       | L3 architecture     | ✅ **Closed** (HSRP on the Distribution)                                                 |
| 2   | A single Core = inter-VLAN routing SPOF           | 🟠       | High availability   | ✅ **Closed** (dual HSRP gateway)                                                        |
| 10  | Statically addressed PCs                          | 🟢       | Scalability         | ✅ **Closed** (DHCP + relay)                                                             |
| 15  | Core = single north-south L3 transit              | 🟠       | High availability   | 📋 Debt: local inter-VLAN survives loss of the Core; external does not                  |
| 15b | Core & HQ-Router with no dedicated management IP  | 🟢       | Ops                 | 📋 Debt: managed in-band via their `/30`s (OSPF-reachable); Loopback0 deferred          |
| 16  | Single-path DHCP relay (helper on the Active only)| 🟢       | Services            | 📋 Debt: deliberate anti-double-relay; no new lease while the Active is down            |
| 8   | Port Security                                     | 🟠       | Security            | 🔜 P3                                                                                    |
| 9   | Voice VLAN 30 with no IP phone                    | 🟢       | Demonstrative       | 🔜 P5                                                                                    |
| 19  | `/30` transit links instead of `/31`             | 🟢       | IPAM                | 📋 Debt: readability/parity choice                                                      |
| 20  | Single OSPF area 0                                | 🟢       | Scalability         | 📋 Debt: multi-area = later hardening                                                   |

## <a id="annexe--captures-de-preuve"></a>Appendix: Evidence captures

**<a id="p-01"></a> [P-01] · Core routed uplinks up**: CORE-SW `show ip interface brief | include 10.0` → Gi1/0/1=10.0.2.1, Gi1/0/2=10.0.3.1, Gi1/0/24=10.0.1.1, all up

![Capture P2-36](../assets/captures/P2/Capture_P2_36.png)

**<a id="p-02"></a> [P-02] · Core OSPF adjacencies (3 neighbors)**: `show ip ospf neighbor` → 2.2.2.2 via 10.0.2.2, 3.3.3.3 via 10.0.3.2, 4.4.4.4 via 10.0.1.2, all `FULL/ -`

![Capture P2-35](../assets/captures/P2/Capture_P2_35.png)

**<a id="p-03"></a> [P-03] · Core ECMP to the VLANs**: `show ip route ospf` → 192.168.10/20/30/99 via 10.0.2.2 **and** 10.0.3.2

![Capture P2-34](../assets/captures/P2/Capture_P2_34.png)

**<a id="p-04"></a> [P-04] · DIST-SW1 adjacency**: `show ip ospf neighbor` → 10.255.255.1 via 10.0.2.1 `FULL/ -`

![Capture P2-32](../assets/captures/P2/Capture_P2_32.png)

**<a id="p-05"></a> [P-05] · DIST-SW1 routes**: `show ip route ospf` → transits via 10.0.2.1

![Capture P2-31](../assets/captures/P2/Capture_P2_31.png)

**<a id="p-06"></a> [P-06] · DIST-SW2 adjacency + routes**: `show ip ospf neighbor` + `route ospf` → 10.255.255.1 via 10.0.3.1

![Capture P2-26](../assets/captures/P2/Capture_P2_26.png)

**<a id="p-07"></a> [P-07] · HQ-Router propagation**: `show ip route ospf` → transits 10.0.2/3 + VLANs via 10.0.1.1

![Capture P2-33](../assets/captures/P2/Capture_P2_33.png)

**<a id="p-08"></a> [P-08] · HQ-Router adjacency (Core↔HQ)**: `show ip ospf neighbor` → 10.255.255.1 via 10.0.1.1

![Capture P2-08](../assets/captures/P2/Capture_P2_08.png)

**<a id="p-09"></a> [P-09] · HSRP split DIST-SW1**: `show standby brief` → Active V10/V30 (Pri 110 P), Standby V20/V99

![Capture P2-11](../assets/captures/P2/Capture_P2_11.png)

**<a id="p-10"></a> [P-10] · HSRP split DIST-SW2 (mirror)**: `show standby brief` → Active V20/V99 (Pri 110 P), Standby V10/V30

![Capture P2-23](../assets/captures/P2/Capture_P2_23.png)

**<a id="p-11"></a> [P-11] · STP root VLAN 10 = DIST1**: `show spanning-tree vlan 10` → `This bridge is the root`

![Capture P2-22](../assets/captures/P2/Capture_P2_22.png)

**<a id="p-12"></a> [P-12] · STP root VLAN 20 = DIST2**: `show spanning-tree vlan 20` → `This bridge is the root`

![Capture P2-20](../assets/captures/P2/Capture_P2_20.png)

**<a id="p-13"></a> [P-13] · STP root VLAN 30 = DIST1**: `show spanning-tree vlan 30` → `This bridge is the root`

![Capture P2-14](../assets/captures/P2/Capture_P2_14.png)

**<a id="p-14"></a> [P-14] · STP root VLAN 99 = DIST2**: `show spanning-tree vlan 99` → `This bridge is the root`

![Capture P2-13](../assets/captures/P2/Capture_P2_13.png)

**<a id="p-15"></a> [P-15] · Failover trigger**: DIST-SW1 `interface vlan 10 / shutdown` → SVI down

![Capture P2-30](../assets/captures/P2/Capture_P2_30.png)

**<a id="p-16"></a> [P-16] · Failover promotion**: DIST-SW2 `show standby brief` → V10 `Active`, Standby `unknown` (DIST1 gone)

![Capture P2-16](../assets/captures/P2/Capture_P2_16.png)

**<a id="p-17"></a> [P-17] · Failover data-plane**: PC `ping -t 192.168.10.1` → 3 timeouts then recovery

![Capture P2-18](../assets/captures/P2/Capture_P2_18.png)

**<a id="p-18"></a> [P-18] · Preempt (transition log)**: DIST-SW1 `interface vlan 10 / no shutdown` → `%HSRP-6-STATECHANGE: Vlan10 Grp 10 Standby -> Active`

![Capture P2-29](../assets/captures/P2/Capture_P2_29.png)

**<a id="p-19"></a> [P-19] · Preempt confirmed**: DIST-SW1 `show standby brief` → V10 `Active`/local, Standby 192.168.10.3

![Capture P2-28](../assets/captures/P2/Capture_P2_28.png)

**<a id="p-20"></a> [P-20] · Centralized DHCP leases**: HQ-ROUTER `show ip dhcp binding` → .10.50–.53 / .20.51–.54, single server

![Capture P2-21](../assets/captures/P2/Capture_P2_21.png)

**<a id="p-21"></a> [P-21] · Routed inter-VLAN**: campus PC `ping .10.51` (0 %) + `ping .20.51` (TTL=127, one hop)

![Capture P2-19](../assets/captures/P2/Capture_P2_19.png)

---

⬅️ [Workflow P1](../P1/WORKFLOW.md) · ⬆️ [Contents](#sommaire) · [Part README](./README.md) · [Project overview](../README.md) · **Next: [Workflow P3](../P3/WORKFLOW.md)** – ASA perimeter, tiered DMZ, NAT/PAT, exact internal routes: no summary that would loop.
