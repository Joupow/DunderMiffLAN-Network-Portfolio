# Part 5: Workflow 

 **Key concepts**: VoIP · CME · TFTP · DHCP Option 150 · voice QoS

- 💻 **Tool**: Cisco Packet Tracer 9.0
- 🏷️ Full addressing plan → [IPAM](../IPAM.md)
- 📄 Part 5 overview → [README P5](./README.md)
- 🎓 **Certification:** CompTIA Network+

---
## <a id="sommaire"></a>Contents

**1. Scope**

- [As-Built Topology](#topologie-as-built)
- [Tiers & equipment](#niveaux--équipements)

**2. Configuration steps**

- [Step 0: Verify ASA routing](#step-0--vérifier-le-routage-asa-aucune-reconfiguration)
- [Step 1: Attach the CME to VLAN 30](#step-1--rattacher-le-cme-au-vlan-30-sur-dist-sw1)
- [Step 2: CME base config](#step-2--config-de-base-du-cme-interface--route-par-défaut)
- [Step 3: DHCP + Option 150](#step-3--dhcp--option-150-autorité-unique-du-vlan-30)
- [Step 4: Call-control + numbers + buttons](#step-4--call-control--numéros--boutons-telephony-service)
- [Step 5: Phone ports (data 10 + voice 30)](#step-5--ports-téléphones-sur-les-access-data-10--voice-30)
- [Step 6: QoS: conditional trust](#step-6--qos--trust-conditionnel-frontière-liée-au-cdp)
- [Phone boot sequence](#séquence-de-boot-dun-poste-où-lécran-lcd-bloque)

**3. Evidence & closure**

- [End-to-end validation](#validation-de-bout-en-bout-gate-final)
- [Troubleshooting (session incidents)](#dépannage-incidents-de-session)
- [Error log & technical debt](#registre-derreurs--dette-technique)
- [Appendix: Evidence captures](#annexe--captures-de-preuve)
# 1. Scope

## <a id="topologie-as-built"></a>As-Built Topology

PT diagram: telephony – CME, voice VLAN, IP phones

![Networ-overview-P5](../assets/network-overview/NO_P5.png)

## <a id="niveaux--équipements"></a>Tiers & equipment

| Role                           | Equipment                                    | Role in this part                                                                                                   |
| ------------------------------ | -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Call agent + DHCP + TFTP       | **CME-Router** (2811) - *new*                | `telephony-service` (SCCP), VLAN 30 DHCP pool, TFTP server (option 150): a single box for all three functions       |
| Distribution (service host)    | **DIST-SW1** (3560-24PS)                     | Hosts the CME on `Fa0/10` (access VLAN 30). Already HSRP Active + STP root for VLAN 30. **Otherwise unchanged.**    |
| Access (endpoints)             | **ACC-SW1 / ACC-SW2**                        | Port `Fa0/5` in data VLAN 10 + voice VLAN 30 + conditional QoS trust                                                 |
| Endpoints                      | **IP Phone 1001 / 1002** (7960) - *new*      | Register over SCCP, a PC plugs in behind (one cable, two VLANs)                                                      |
| Perimeter (control)            | **ASA / HQ-Router**                          | Not reconfigured: we only **verify** the safe `/20` summary and the absence of a VLAN 30 pool on the HQ             |

The whole P1/P2 campus, the P3 perimeter and the P4 datacenter are **unchanged**; P5 only adds the CME, two phones and the ports' voice config.

# 2. Configuration steps

### <a id="step-0--vérifier-le-routage-asa-aucune-reconfiguration"></a>Step 0: Verify ASA routing (no reconfiguration)

**Intent:** confirm the P3 `/20` summary is still safe. Verification only.

```cisco
! ===== ASA =====

show route | include inside
! Expected: S 10.0.0.0 255.255.240.0  (a single /20 summary; NO /8, NO /16)

! ===== HQ-Router =====

show run | include Null0
! Expected: ip route 10.0.0.0 255.255.240.0 Null0 254  (lock for the /20 holes)
```

> 📷 **[P-09](#p-09)** ASA `show route`: `S 10.0.0.0 255.255.240.0`, no `/8`/`/16`.

---

### <a id="step-1--rattacher-le-cme-au-vlan-30-sur-dist-sw1"></a>Step 1: Attach the CME to VLAN 30 (on DIST-SW1)

**Intent:** plug the CME into the switch that is STP root + HSRP Active for VLAN 30.

```cisco
! ===== DIST-SW1 =====

enable
configure terminal

interface FastEthernet0/10
 description CME-Router_VLAN30
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
 
end
write memory
```

> ⚠️ **Incident I-1:** the port can stay `Down` **despite** this config if the far-end interface (CME `Fa0/0`) is in `shutdown`. Configuring a port does not create the physical link.

**Check:** `show interfaces status Fa0/10` → `connected`, VLAN `30`. Placement confirmed by `show standby brief` → `Vl30 … Active` on DIST-SW1.

> 📷 **[P-08](#p-08)** DIST-SW1 `show standby brief`: `Vl30 30 110 P Active`.

---

### <a id="step-2--config-de-base-du-cme-interface--route-par-défaut"></a>Step 2: CME base config (interface + default route)

**Intent:** service IP on `Fa0/0`, default route to the HSRP VIP.

```cisco
! ===== CME-Router =====

enable
configure terminal

hostname CME-Router

interface FastEthernet0/0
 description VLAN30_VoIP
 ip address 192.168.30.254 255.255.255.0
 no shutdown                              ! ← router interfaces are born 
SHUTDOWN
exit

ip route 0.0.0.0 0.0.0.0 192.168.30.1     ! gateway = HSRP VIP (survives the DIST1→DIST2 failover)

end
write memory
```

**Check:**

```cisco
show ip interface brief      ! Fa0/0 = 192.168.30.254 up/up
ping 192.168.30.1            ! reaches the HSRP VIP (DIST-SW1 alive)
show arp                     ! .1 resolved = VLAN 30 present on the Distribution
```

> 📷 **[P-12a](#p-12a)** `Fa0/0 .254 up/up` · **[P-12b](#p-12b)** `ping .1` = 5/5 · **[P-12c](#p-12c)** `show arp` (`.1` resolved).

---

### <a id="step-3--dhcp--option-150-autorité-unique-du-vlan-30"></a>Step 3: DHCP + Option 150 (SOLE authority for VLAN 30)

**Intent:** voice pool confined to `.50-.99`, option 150 = the CME's own IP.

```cisco
! ===== CME-Router =====

configure terminal

ip dhcp excluded-address 192.168.30.1 192.168.30.49
ip dhcp excluded-address 192.168.30.100 192.168.30.254

ip dhcp pool VOIP_PHONES
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1        ! = HSRP VIP (not the CME's IP) → the phones' gateway survives a failover
 option 150 ip 192.168.30.254       ! TFTP address = the CME's OWN IP
 
end
write memory
```

> **What I do NOT do** (guarantees "a single DHCP authority on VLAN 30"): no `ip helper-address` on the DISTs' SVI 30 (the CME is **in** VLAN 30, direct broadcast); no `ip dhcp pool 192.168.30.x` on the HQ-Router (otherwise 2 servers on the same domain → conflict).

**Check (once the phones have booted):**

```cisco
show ip dhcp pool            ! pool VOIP_PHONES, exclusions
show ip dhcp binding         ! leases .50 / .51, "Automatic", all from the CME
```

Proof there's no 2nd server, on the HQ-Router side: `show run | section dhcp` = `VLAN10`/`VLAN20` pools only, no `192.168.30.x`.

> 📷 **[P-06](#p-06)** leases `.50`/`.51` · **[P-06b](#p-06b)** pool `VOIP_PHONES` · **[P-07](#p-07)** HQ-Router: pools 10/20 only.

---

### <a id="step-4--call-control--numéros--boutons-telephony-service"></a>Step 4: Call-control + numbers + buttons (telephony-service)

**Intent:** declare the DNs, the ephones, and **bind** a button to a DN: the step whose omission blocks the phone (Incident I-2).

```cisco
! ===== CME-Router =====

configure terminal

telephony-service
 max-ephones 10
 max-dn 10
 ip source-address 192.168.30.254 port 2000    ! THE CME's IP, never the VIP .1
 auto assign 1 to 10                            ! MUST exist BEFORE the phones' 1st registration
 exit
 
! --- Directory numbers (the extensions) ---
ephone-dn 1
 number 1001
 exit
ephone-dn 2
 number 1002
 exit
 
! --- ephones + button→DN BINDING (the forgotten step) ---
ephone 1
 mac-address 0001.42BC.87D4
 button 1:1                                     ! phone button 1 → ephone-dn 1 (1001)
 exit
ephone 2
 mac-address 0001.C925.A973
 button 1:2                                     ! phone button 1 → ephone-dn 2 (1002)
 exit
telephony-service
 reset all                                      ! forces the phones to re-register cleanly
 
end
write memory
```

> ⚠️ **`ip source-address` trap:** setting it to the VIP `.1` → the phone opens TCP 2000 to a Distribution switch that doesn't do SCCP → never registers. SCCP listens on the CME's real IP (`.254`).
> 
> ⚠️ **`button` trap (I-2):** without `button 1:X`, `show ephone` shows `UNREGISTERED / IP:0.0.0.0 / button 1: dn CH1 DOWN` and the screen loops on "Configuring CM List". `auto assign` does **not** recover ephones already auto-created without a button → **explicit** assignment mandatory, then `reset all`.

**Check:**

```cisco
show run | section ephone    ! ephone-dn 1001/1002 + ephone 1/2 with "button 1:1" / "button 1:2"
show ephone                  ! REGISTERED in SCCP, IP .50/.51, "button 1: dn 1 number 1001 ... IDLE"
```

> 📷 **[P-05](#p-05)** `show run | section ephone` (button 1:1/1:2, fixes I-2) · **[P-04](#p-04)** `show ephone` (`REGISTERED in SCCP`).

---

### <a id="step-5--ports-téléphones-sur-les-access-data-10--voice-30"></a>Step 5: Phone ports on the Access switches (data 10 + voice 30)

**Intent:** one cable, two VLANs: PC in data 10 (untagged), phone in voice 30 (802.1Q-tagged via CDP).

> **Quarantine note (Incident I-3):** `Fa0/5` was found in VLAN 998 + `shutdown` (quarantine applied wider than the `0/20-24` plan). No separate step: `switchport access vlan 10` **overwrites** the 998 and `no shutdown` cancels the quarantine.

```cisco
! ===== ACC-SW1 (same as ACC-SW2) =====

configure terminal

interface FastEthernet0/5
 description IP-Phone
 switchport mode access
 switchport access vlan 10          ! data: REPLACES VLAN 998 (PC behind the phone, untagged)
 switchport voice vlan 30           ! voice (phone, 802.1Q 30-tagged via CDP)
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown                        ! wakes the port (was shut in quarantine)
 
end
write memory
```

> ⚠️ **Do NOT copy `no cdp enable` here.** A **phone** port needs CDP **on**: it's what advertises the voice-VLAN to the phone and arms `trust device cisco-phone` (Step 6). CDP off = phone stuck on "Configuring vlan".
>
> ⚠️ **Without `switchport voice vlan 30`**: no voice-VLAN advertisement → screen stuck on "Configuring vlan", empty DHCP binding. The symptom appears **three steps after** the cause.

**Check:**

```cisco
show interfaces Fa0/5 switchport | include Access|Voice   ! Access VLAN 10, Voice VLAN 30
show vlan brief                                           ! Fa0/5 in 10 and 30, no longer in 998
```

> 📷 **[P-10](#p-10)** ACC-SW1 `switchport` (Access 10 / Voice 30) · **[P-10b](#p-10b)** `vlan brief` (D6 proof) · **[P-10c](#p-10c)** ACC-SW2 symmetry.

---

### <a id="step-6--qos--trust-conditionnel-frontière-liée-au-cdp"></a>Step 6: QoS: conditional trust (CDP-bound boundary)

**Intent:** trust the EF DSCP only if a real Cisco phone is detected via CDP.

```cisco
! ===== ACC-SW1 (same as ACC-SW2) =====

configure terminal

mls qos                              ! global QoS engine: often-forgotten PREREQUISITE
interface FastEthernet0/5
 mls qos trust dscp                  ! without this: no trust type to apply
 mls qos trust device cisco-phone    ! EF DSCP trusted ONLY if a real Cisco phone is detected via CDP
 
end
write memory
```

| Scenario | `trust dscp` alone | `trust dscp` + `trust device cisco-phone` |
|---|---|---|
| Real Cisco phone | Trust DSCP ✅ | Trust DSCP ✅ (after CDP) |
| PC plugged in directly | Trust DSCP ❌ (spoofable) | **Not trusted** ✅ |
| PC behind the phone | Trust DSCP ❌ (spoofable) | **Not trusted** ✅ |

**Check:** `show mls qos interface Fa0/5` → `trust device: cisco-phone`, `trust state: trust dscp`.

> 📷 **[P-11](#p-11)** ACC-SW1 `show mls qos interface Fa0/5`: `trust device: cisco-phone`.

## <a id="séquence-de-boot-dun-poste-où-lécran-lcd-bloque"></a>Phone boot sequence (where the LCD screen gets stuck)

| LCD screen | Meaning | Cause if stuck here |
|---|---|---|
| `Configuring vlan` | Waits for the voice-VLAN advertisement over CDP | `switchport voice vlan 30` missing, or CDP off (Step 5) |
| `Configuring IP` | DHCP in progress | Pool / Option 150 (Step 3) or VLAN 30 reachability |
| `Updating` | Config download over TFTP | Wrong Option 150, or TFTP asymmetry (debt D2, prod ref) |
| `Configuring CM List` | SCCP session opening (TCP 2000) | `ip source-address` = VIP instead of the CME IP **OR** missing `button` (I-2) |

# 3. Evidence & closure

## <a id="validation-de-bout-en-bout-gate-final"></a>End-to-end validation

| Domain         | Check                    | Key command                                             | Expected                                          | Evidence                                          |
| -------------- | ------------------------ | ------------------------------------------------------- | ------------------------------------------------- | ------------------------------------------------- |
| 🧭 Routing     | P3 inheritance (safe summary) | ASA `show route \| include inside`                 | `S 10.0.0.0/20`, no `/8`/`/16`                    | [P-09](#p-09)                                     |
| 📞 CME service | Service placement        | DIST-SW1 `show standby brief`                           | `Vl30 … Active`                                   | [P-08](#p-08)                                     |
| 📞 CME service | CME link                 | CME `show ip int brief` + `ping .1` + `show arp`        | `Fa0/0 .254 up/up`, `.1` resolved                 | [P-12a](#p-12a), [P-12b](#p-12b), [P-12c](#p-12c) |
| 📞 CME service | Single-authority DHCP    | CME `show ip dhcp binding` + HQ `show run \| sect dhcp` | `.50`/`.51` leases (CME only), no pool 30 on HQ   | [P-06](#p-06), [P-07](#p-07)                      |
| 🎚️ Voice & QoS | Voice VLAN + tag         | ACC `show int Fa0/5 switchport`                         | Access 10 / Voice 30                              | [P-10](#p-10), [P-10c](#p-10c)                    |
| 🎚️ Voice & QoS | Conditional QoS trust    | ACC `show mls qos interface Fa0/5`                      | `trust device: cisco-phone`                       | [P-11](#p-11)                                     |
| 📞 CME service | SCCP registration        | CME `show ephone` + `show run \| section ephone`        | `REGISTERED in SCCP`, `button 1:1`/`1:2`          | [P-04](#p-04), [P-05](#p-05)                      |
| ☎️ Call       | End-to-end 1001 → 1002   | phone 1001 calls 1002                                   | `Ring Out` → `ringing` → `Connected`              | [P-01](#p-01), [P-02](#p-02), [P-03](#p-03)       |

## <a id="dépannage-incidents-de-session"></a>Troubleshooting (session incidents)

> Incidents actually encountered, with the diagnostic that caught each one. **Not** debts; each fixed the same day.

| # | Symptom | Root cause | Diagnosis | Fix |
|---|---|---|---|---|
| **I-1** | DIST-SW1 `Fa0/10` stays `Down` even though the CLI was typed | CME's `Fa0/0` interface left in `shutdown` (**router** interfaces are born shut) | `show ip interface brief` on the CME | `no shutdown` on CME `Fa0/0`: the link comes up on both sides |
| **I-2** | Phones stuck on "Configuring CM List" [P-14](#p-14); `show ephone` = `UNREGISTERED`, `IP:0.0.0.0` [P-13](#p-13) | ephones auto-created **with no `button` line** [P-15](#p-15) → no DN presented | `show run \| section ephone` → no `button 1:X` | Add explicit `button 1:1` / `1:2`, then `reset all`: fixed = [P-05](#p-05) |
| **I-3** | `Fa0/5` found in VLAN 998 + `shutdown` | Quarantine applied wider (`0/5-24`) than the plan (`0/20-24`) | `show vlan brief` → `Fa0/5` under 998 | Reassigned to `access 10` + `voice 30`; no impact: see debt **D6** |

> **Design point clarified (not an incident):** "centralize everything on a single DHCP at the HQ-Router" ruled out. A global DHCP = SPOF for all addressing (data + voice) on the most exposed box. Voice stays local to the CME so it can boot without depending on campus routing. Decision: **one authority per domain**.

**Reference commands:**

```cisco
! CME       : show ip interface brief · show ip dhcp binding · show ip dhcp pool
!             show run | section telephony-service · show run | section ephone · show ephone
! Access    : show interfaces Fa0/5 switchport · show vlan brief · show mls qos interface Fa0/5
! Dist      : show standby brief          (VLAN 30 = DIST-SW1 Active)
! ASA / HQ  : show route | include inside · show run | section dhcp  (pools 10/20 only)
```

---

## <a id="registre-derreurs--dette-technique"></a>Error log & technical debt

**Design decisions & pitfalls navigated**

| Ref. | Decision taken (and the pitfall it avoids)                                                           | Status |
| ---- | ---------------------------------------------------------------------------------------------------- | ------ |
| A1   | CME on DIST-SW1 (Active + VLAN 30 root), not on an Access → avoids the hardware SPOF                 | ✅      |
| A2   | Conditional QoS trust: `trust device cisco-phone` → EF DSCP trusted only for a real Cisco phone      | ✅      |
| B1   | ASA `/20` summary audited safe: no `/8`/`/16`, Null0 lock set in P3                                  | ✅      |
| B2   | DHCP pool restricted to `.50-.99` (exclusions `.1-.49` and `.100-.254`)                              | ✅      |
| B3   | PortFast = access port only (not a trunk anti-flap)                                                  | ✅      |
| B4   | No `ip helper-address` on VLAN 30: the CME is in the VLAN, direct broadcast                          | ✅      |

**Session incidents: resolved**

| Ref. | Description                                                          | Severity | Domain   | Status                               |
| ---- | -------------------------------------------------------------------- | -------- | -------- | ------------------------------------ |
| I-1  | CME `Fa0/0` left in `shutdown` → link down                          | 🟠       | Physical | ✅ `no shutdown`                      |
| I-2  | Missing `button` (button→DN) → phones UNREGISTERED (the central pitfall) | 🟠  | SCCP     | ✅ `button 1:1` / `1:2` + `reset all` |
| I-3  | `Fa0/5` parked in 998 (quarantine too wide)                         | 🟢       | Plan     | ✅ Reassigned; overreach noted in D6  |

**Accepted debts: documented, not fixed**

| Ref. | Debt                                                                                                   | Severity | Domain              | Status                                                     |
| ---- | ------------------------------------------------------------------------------------------------------ | -------- | ------------------- | ---------------------------------------------------------- |
| D1   | CME on DIST-SW1 = residual SPOF (Loopback + `ip tftp source-interface Loopback0` unsupported in PT)     | 🟠       | High availability   | 📋 PT debt: prod = service on Loopback                     |
| D2   | Single TFTP service, no backup                                                                         | 🟢       | Services            | 📋 Documented                                              |
| D3   | VLAN 30 extended in L2 → trunk flap (outside the PortFast scope)                                        | 🟢       | STP                 | 📋 Mitigated by RSTP                                       |
| D4   | Phones 1003/1004 not deployed (2 of 4)                                                                 | 🟢       | Scope               | 🔜 Extension possible                                      |
| D5   | Summaries `192.168.0.0/16` and `172.16.0.0/20` = residual exposure                                      | 🟠       | Routing             | 📋 OSPF on the ASA = definitive fix                        |
| D6   | Quarantine 998 applied to `Fa0/5-24` instead of `0/20-24`                                               | 🟢       | Plan                | 📋 Deviation: audit `show interfaces status \| include 998` |

> **Prod ref (the "Updating" pitfall outside PT):** CME service IP on `Loopback0` + `ip tftp source-interface Loopback0`. 
> 
> Without it, the router may answer TFTP from its egress physical interface's IP; a phone expecting `.254` rejects the asymmetric packet and stays stuck on "Updating".

## <a id="annexe--captures-de-preuve"></a>Appendix: Evidence captures

**<a id="p-01"></a> [P-01] · Call placed**: Phone 1001 `To: 1002 / Ring Out`

![Capture P5-07](../assets/captures/P5/Capture_P5_07.png)

**<a id="p-02"></a> [P-02] · Call received (SCCP)**: Phone 1002 `From: 1001 / ringing`

![Capture P5-06](../assets/captures/P5/Capture_P5_06.png)

**<a id="p-03"></a> [P-03] · Media established**: Phone 1002 `Connected`

![Capture P5-05](../assets/captures/P5/Capture_P5_05.png)

**<a id="p-04"></a> [P-04] · SCCP registration**: CME `show ephone`: `REGISTERED in SCCP`, IP `.50`/`.51`, `button 1: dn 1/2 … IDLE`

![Capture P5-04](../assets/captures/P5/Capture_P5_04.png)

**<a id="p-05"></a> [P-05] · button→DN (fixes I-2) + telephony-service**: CME `show run | section ephone`: `ip source-address .254`, `max 10`, `ephone 1/2` with `button 1:1`/`1:2`, `type 7960`

![Capture P5-27](../assets/captures/P5/Capture_P5_27.png)

**<a id="p-06"></a> [P-06] · DHCP leases**: CME `show ip dhcp binding`: `.50`/`.51` `Automatic`

![Capture P5-18](../assets/captures/P5/Capture_P5_18.png)

**<a id="p-06b"></a> [P-06b] · DHCP pool**: CME `show ip dhcp pool`: `VOIP_PHONES`

![Capture P5-19](../assets/captures/P5/Capture_P5_19.png)

**<a id="p-07"></a> [P-07] · No 2nd DHCP server**: HQ-Router `show run | section dhcp`: `VLAN10`/`VLAN20` only

![Capture P5-03](../assets/captures/P5/Capture_P5_03.png)

**<a id="p-08"></a> [P-08] · Service placement**: DIST-SW1 `show standby brief`: `Vl30 30 110 P Active`

![Capture P5-01](../assets/captures/P5/Capture_P5_01.png)

**<a id="p-09"></a> [P-09] · Summary audit closed**: ASA `show route`: `S 10.0.0.0 255.255.240.0`, no `/8`/`/16`

![Capture P5-02](../assets/captures/P5/Capture_P5_02.png)

**<a id="p-10"></a> [P-10] · Voice VLAN (ACC-SW1)**: `show interfaces Fa0/5 switchport`: `Access 10 / Voice 30`

![Capture P5-21](../assets/captures/P5/Capture_P5_21.png)

**<a id="p-10b"></a> [P-10b] · VLAN brief + D6 proof**: ACC-SW1 `show vlan brief`: `Fa0/5` in 10 and 30

![Capture P5-15](../assets/captures/P5/Capture_P5_15.png)

**<a id="p-10c"></a> [P-10c] · 2nd phone symmetry (ACC-SW2)**: `show interfaces Fa0/5 switchport`: `Access 10 / Voice 30`

![Capture P5-20](../assets/captures/P5/Capture_P5_20.png)

**<a id="p-11"></a> [P-11] · Conditional QoS trust**: ACC-SW1 `show mls qos interface Fa0/5`: `trust device: cisco-phone`

![Capture P5-14](../assets/captures/P5/Capture_P5_14.png)

**<a id="p-12a"></a> [P-12a] · CME link (interface)**: CME `show ip interface brief`: `Fa0/0 .254 up/up`

![Capture P5-25](../assets/captures/P5/Capture_P5_25.png)

**<a id="p-12b"></a> [P-12b] · CME link (ping)**: CME `ping 192.168.30.1` = 5/5

![Capture P5-23](../assets/captures/P5/Capture_P5_23.png)

**<a id="p-12c"></a> [P-12c] · CME link (ARP)**: CME `show arp`: `.1` and `.254` resolved

![Capture P5-22](../assets/captures/P5/Capture_P5_22.png)

**Incidents (state *before* fix: never in validation)**

**<a id="p-13"></a> [P-13] · I-2 symptom (SCCP)**: CME `show ephone`: `UNREGISTERED`, `IP:0.0.0.0`, `button 1: dn CH1 DOWN`

![Capture P5-11](../assets/captures/P5/Capture_P5_11.png)

**<a id="p-14"></a> [P-14] · I-2 symptom (LCD)**: Phone stuck on `Configuring CM List`

![Capture P5-13](../assets/captures/P5/Capture_P5_13.png)

**<a id="p-15"></a> [P-15] · I-2 cause**: CME `show run | section ephone` **without** a `button` line

![Capture P5-10](../assets/captures/P5/Capture_P5_10.png)

> **Dropped at triage:** `Captures_P5_26` (duplicate of 27), `_17` (duplicate of 10), `_24` (degraded 4/5 ping), `_16` (redundant with 21), `_12` (truncated telephony-service), `_8`/`_9` (idle phones).

---

⬅️ [Workflow P4](../P4/WORKFLOW.md) · ⬆️ [Contents](#sommaire) · [Part README](./README.md) · [Project overview](../README.md) · **Next: [Workflow P6](../P6/WORKFLOW.md)** – Wi-Fi – WLC, APs, HSRPv2 VLAN 300, DHCP 300 on DIST-SW1.
