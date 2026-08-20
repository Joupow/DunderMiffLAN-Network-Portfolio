# Part 3: Workflow 

**Key concepts**: ASA, DMZ, NAT/PAT & filtering

- 💻 **Tool**: Cisco Packet Tracer 9.0
- 🏷️ Full addressing plan → [IPAM](../IPAM.md)
- 📄 Part 3 overview → [README P3](./README.md)
- 🎓 **Certification:** CompTIA Network+
## <a id="sommaire"></a>Contents

**1. Scope**

- [As-Built Topology](#topologie-as-built)
- [Tiers & equipment](#niveaux--équipements)
- [PT / ASA 9.6 command limits](#limites-de-commande-pt--asa-96-source-unique)

**2. Configuration steps**

- [Step 1: Equipment & cabling](#étape-1--équipements--câblage)
- [Step 2: ASA interfaces + security-levels](#étape-2--interfaces-asa--security-levels)
- [Step 3: HQ-Router: default origination](#étape-3--hq-router--lien-inside--origination-du-défaut-le-déblocage)
- [Step 4: ISP-Router: Internet + external test](#étape-4--isp-router--internet--segment-de-test-externe)
- [Step 5: ASA routing](#étape-5--routage-asa-prouver-la-joignabilité-avant-toute-acl)
- [Step 6: NAT / PAT](#étape-6--nat--pat)
- [Step 7: Stateful ICMP inspection](#étape-7--inspection-icmp-stateful)
- [Step 8: OUTSIDE-IN ACL](#étape-8--acl-outside-in-deny-par-défaut-exceptions-chirurgicales)
- [Step 9: INSIDE-FORCED-PROXY ACL](#étape-9--acl-inside-forced-proxy-le-permit-final-est-obligatoire)
- [Step 10: DMZ-RESTRICT ACL](#étape-10--acl-dmz-restrict-deny-implicite--pas-de-permit-final)
- [Step 11: Hardening: port-security + SPAN](#étape-11--durcissement--port-security-access--span-core)

**3. Evidence & closure**

- [End-to-end validation](#validation-de-bout-en-bout-gate-final)
- [Troubleshooting (session incidents)](#dépannage-incidents-de-session)
- [Error log & technical debt](#registre-derreurs--dette-technique)
- [Appendix: Evidence captures](#annexe--captures-de-preuve)

# 1. Scope

## <a id="topologie-as-built"></a>As-Built Topology

PT diagram: perimeter & security – ASA, DMZ, ISP peering, IDS

![Networ-overview-P3](../assets/network-overview/NO_P3.png)

## <a id="niveaux--équipements"></a>Tiers & equipment

| Role               | Equipment                                           | Role in this part                                                                                    |
| ------------------ | --------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Edge firewall      | **ASA-EDGE** (ASA 5506-X): *new*                    | 3 zones (outside 0 / dmz 50 / inside 100); NAT/PAT; 3 ACLs; stateful ICMP inspection                 |
| Internet           | **ISP-Router** (2911) - *new*                       | Simulated Internet: loopback `8.8.8.8`; outside `/30`; carries the external test PC                  |
| Services (routing) | **HQ-Router** (ISR 2911)                            | Gains the ASA-inside `/30`, the default route, and **originates `0.0.0.0/0` into OSPF**; Null0 locks |
| DMZ switch         | **DMZ-SW** (2960) - *new*                           | L2 for the two DMZ servers                                                                           |
| DMZ servers        | **WEB-PUBLIC** `.10` / **PROXY** `.20` - *new*      | Published front / forced-proxy egress point                                                          |
| Detection          | **IDS-Sensor** (`.99.20`) - *new*                   | SPAN destination for the Core's edge uplink (passive)                                                |
| Access             | 4× Catalyst **2960**                                | **Port-security closed**: sticky, `maximum 2`, `violation restrict`                                  |

The whole P1/P2 campus is **unchanged**: P3 only bolts the perimeter onto the existing HQ-Router (which keeps `Gi0/0`=Core; the ASA lands on `Gi0/1`, which was free).

### <a id="limites-de-commande-pt--asa-96-source-unique"></a>PT / ASA 9.6 command limits 

| Standard command | PT behavior | Use instead |
|---|---|---|
| `show nameif` | ✗ invalid | `show running-config` (read nameif in the interface blocks) |
| `show access-list NAME` | ✗ invalid | `show access-list` (scroll to the ACL) |
| `show service-policy` | ✗ invalid | `show running-config policy-map` |
| `no access-list NAME` (whole ACL) | ✗ incomplete | remove line by line: `no access-list NAME extended <full rule>` |
| `nat … static X service tcp 80 80` | ✗ invalid at `service` | 1:1 static NAT + ACL filter at `:80` |
| `access-list … log` | ✗ invalid at `log` | omit: SIEM = P8 item |
| `… time-exceeded` | ✗ unsupported | omit: traceroute only; PMTUD preserved by `unreachable` |
| inline `! comment` after a command | ✗ invalid at `!` | comments on their own line only |

# 2. Configuration steps

The firewall is built inside-out: reachability, then translation, then filtering. Step 3 (default origination) is the real unlock, to be done before the ASA's routes, otherwise the ASA has a next-hop that no host reaches.

---
### <a id="étape-1--équipements--câblage"></a>Step 1: Equipment & cabling

**Intent:** add the perimeter; the campus is untouched.

- **ASA `Gi1/1`** ↔ **ISP `Gi0/0`** (`203.0.113.0/30`)
- **ASA `Gi1/2`** ↔ **HQ `Gi0/1`** (`192.168.200.0/30`): HQ `Gi0/0` stays on the Core
- **ASA `Gi1/3`** ↔ **DMZ-SW `Gi0/1`**; DMZ-SW `Fa0/1`/`Fa0/2` → WEB-PUBLIC / PROXY
- **ISP `Gi0/1`** ↔ **PC-EXTERIEUR** (`198.51.100.0/24`)
- **CORE `Gi1/0/5`** ↔ **IDS-Sensor** (SPAN destination, VLAN 99)

Servers (Desktop → IP Configuration): 

- WEB-PUBLIC `172.16.0.10/24` GW `.1` 
- PROXY `172.16.0.20/24` GW `.1` 
- IDS-Sensor `192.168.99.20/24` GW `192.168.99.1`
- PC-EXTERIEUR `198.51.100.10/24` GW `198.51.100.1`.

> ⚠️ On WEB-PUBLIC and PROXY: **Services → HTTP → On** (to tell an allowed flow that renders a page apart from a denied one).

---

### <a id="étape-2--interfaces-asa--security-levels"></a>Step 2: ASA interfaces + security-levels

**Intent:** without `nameif`, the ASA treats a port as nonexistent: no ACL or NAT can reference it.

```cisco
enable
configure terminal

interface GigabitEthernet1/1
 nameif outside
 security-level 0
 ip address 203.0.113.2 255.255.255.252
 no shutdown
 
interface GigabitEthernet1/2
 nameif inside
 security-level 100
 ip address 192.168.200.1 255.255.255.252
 no shutdown
 
interface GigabitEthernet1/3
 nameif dmz
 security-level 50
 ip address 172.16.0.1 255.255.255.0
 no shutdown
 
end
write memory
```

> **Asymmetry rule:** higher→lower passes by default (inside→outside OK); lower→higher is blocked by default (outside→inside needs an explicit ACL).

**Validation:** `show running-config` → read nameif/security-level in each interface block (`show nameif` invalid in PT). `show interface ip brief` → `Gi1/1`, `Gi1/2`, `Gi1/3` all `up up`.

---

### <a id="étape-3--hq-router--lien-inside--origination-du-défaut-le-déblocage"></a>Step 3: HQ-Router: inside link + default origination (THE unlock)

**Intent:** push the default route into OSPF and lock the empty summarized space.

```cisco
enable
configure terminal

interface GigabitEthernet0/1
 ip address 192.168.200.2 255.255.255.252
 no shutdown
 
ip route 0.0.0.0 0.0.0.0 192.168.200.1
ip route 192.168.0.0 255.255.0.0 Null0 254
ip route 10.0.0.0 255.255.240.0 Null0 254

router ospf 1
 default-information originate
 
end
write memory
```

> **Why the two Null0 locks.**
> 
> - The ASA summarizes the inside as `10.0.0.0/20` **and** `192.168.0.0/16`. 
> - A summary is only safe if every empty block it covers has a floating black hole (AD 254). 
> 
> Without the `/16` lock, a packet to `192.168.150.1` (empty VLAN) bounces ASA↔HQ until the TTL expires; 
> 
> without the `/20`, same on `10.0.14.x`. LPM (Longest Prefix Match) means a real OSPF prefix always wins; Null0 only catches the holes.

**Validation:**

```cisco
show ip route static | include 0.0.0.0    ! S* 0.0.0.0/0 via 192.168.200.1
```

Then on **DIST-SW1** and **CORE-SW**: `show ip route ospf | include 0.0.0.0` → `O*E2 0.0.0.0/0`. **This is the proof that origination works.**

> 📷 Proof of the originated default is consolidated in the matrix (group A). See appendix.

---

### <a id="étape-4--isp-router--internet--segment-de-test-externe"></a>Step 4: ISP-Router: Internet + external test segment

**Intent:** simulate the Internet (loopback `8.8.8.8`) and the external PC.

```cisco
enable
configure terminal

hostname ISP-Router

interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.252
 no shutdown
 
interface GigabitEthernet0/1
 ip address 198.51.100.1 255.255.255.0
 no shutdown
 
interface Loopback0
 ip address 8.8.8.8 255.255.255.255
 
end
write memory
```

> No return route needed for NAT'd traffic: internal hosts egress as `203.0.113.2` (connected to the ISP). PC-EXTERIEUR reaches the server published on `203.0.113.2`, the ASA's outside interface.

---

### <a id="étape-5--routage-asa-prouver-la-joignabilité-avant-toute-acl"></a>Step 5: ASA routing (prove reachability BEFORE any ACL)

**Intent:** the ASA uses `route <nameif>`, never `ip route`. Contiguous transits → a single `/20` summary, locked by the Null0 from step 3.

```cisco
configure terminal

route outside 0.0.0.0 0.0.0.0 203.0.113.1
route inside 192.168.0.0 255.255.0.0 192.168.200.2
route inside 10.0.0.0 255.255.240.0 192.168.200.2

end
write memory
```

**Validation (no-ACL window: everything must pass):**

```cisco
show route                       ! Gateway of last resort = 203.0.113.1 ; S* 0.0.0.0/0
ping 172.16.0.10                 ! WEB-PUBLIC -> !!!!!
ping 172.16.0.20                 ! PROXY      -> !!!!!
```

Then from **PC1**: `ping 8.8.8.8` → reply after one ARP loss. If PC1 still fails here, the fault is in steps 3–5, not NAT.

---

### <a id="étape-6--nat--pat"></a>Step 6: NAT / PAT

**Intent:** two dynamic PATs (inside, dmz) + one 1:1 static publication.

```cisco
configure terminal

object network OBJ-INSIDE-PAT
 subnet 192.168.0.0 255.255.0.0
 nat (inside,outside) dynamic interface
 
object network OBJ-DMZ-PAT
 subnet 172.16.0.0 255.255.255.0
 nat (dmz,outside) dynamic interface
 
object network OBJ-WEB-PUBLIC
 host 172.16.0.10
 nat (dmz,outside) static 203.0.113.2
 
end
write memory
```

> On a bare outside `/30`, no spare public IP: WEB-PUBLIC is published on the interface address `.2`, the same one as PAT overload. 
> 
> This shared-address publication is the root of the inbound-rendering limitation (debt #22). Since `service tcp 80 80` is unsupported, OUTSIDE-IN (step 8) filters the inbound at `:80`.

**Validation:** from **PC1** `ping 8.8.8.8`, then on the ASA `show xlate` → a **dynamic** entry: `ICMP PAT from inside:192.168.10.50 to outside:203.0.113.2 flags i`. The static entry shows permanently and proves nothing on its own: the **dynamic** entry is the proof.

> ⚠️ If you change a NAT mapping, run `clear xlate`: the old translation is cached.
> 
> 📷 **[P-05](#p-05)** `show xlate` (dynamic `flags i` + static `flags s`).

---

### <a id="étape-7--inspection-icmp-stateful"></a>Step 7: Stateful ICMP inspection

**Intent:** TCP/UDP are tracked by default; ICMP is not. Without inspection, an echo-reply on outside (level 0) is an unsolicited inbound, dropped.

```cisco
configure terminal

class-map inspection_default
 match default-inspection-traffic
 
policy-map global_policy
 class inspection_default
  inspect icmp
  
service-policy global_policy global

end
write memory
```

**Validation:** `show running-config policy-map` → `inspect icmp` under `policy-map global_policy` (`show service-policy` invalid in PT). This step is what creates the ICMP xlate from step 6.

---

### <a id="étape-8--acl-outside-in-deny-par-défaut-exceptions-chirurgicales"></a>Step 8: OUTSIDE-IN ACL (deny by default, surgical exceptions)

**Intent:** untrusted zone: deny everything, open only HTTP→WEB-PUBLIC and the necessary ICMP.

```cisco
configure terminal

access-list OUTSIDE-IN extended deny tcp any any eq 23
access-list OUTSIDE-IN extended deny tcp any any eq 22
access-list OUTSIDE-IN extended deny tcp any any eq 445
access-list OUTSIDE-IN extended deny icmp any any echo
access-list OUTSIDE-IN extended permit icmp any any echo-reply
access-list OUTSIDE-IN extended permit icmp any any unreachable
access-list OUTSIDE-IN extended permit tcp any host 172.16.0.10 eq www
access-group OUTSIDE-IN in interface outside

end
write memory
```

> **Surgical ICMP:** `deny icmp any any` would break PMTUD (blocks Type-3). Block echo (stops ping sweeps), keep echo-reply and unreachable. 
> 
> The HTTP permit references the **real** post-NAT IP (`172.16.0.10`), never the public one. `time-exceeded` (Type 11) unsupported in PT → omitted (debt #24).

**Validation: non-regression first:** re-run PC1 `ping 8.8.8.8`. It must **survive** (the `permit … echo-reply` counter climbs). If it breaks now, it's **this** echo-reply line that's missing, not one added later.

---

### <a id="étape-9--acl-inside-forced-proxy-le-permit-final-est-obligatoire"></a>Step 9: INSIDE-FORCED-PROXY ACL (the final permit is MANDATORY)

**Intent:** permit proxy first, then deny direct 80/443, DNS allowed, final `permit ip any any`.

```cisco
configure terminal

access-list INSIDE-FORCED-PROXY extended permit tcp 192.168.0.0 255.255.0.0 host 172.16.0.20 eq www
access-list INSIDE-FORCED-PROXY extended permit udp 192.168.0.0 255.255.0.0 any eq domain
access-list INSIDE-FORCED-PROXY extended permit tcp 192.168.0.0 255.255.0.0 any eq domain
access-list INSIDE-FORCED-PROXY extended deny tcp 192.168.0.0 255.255.0.0 any eq www
access-list INSIDE-FORCED-PROXY extended deny tcp 192.168.0.0 255.255.0.0 any eq 443
access-list INSIDE-FORCED-PROXY extended permit ip any any
access-group INSIDE-FORCED-PROXY in interface inside

end
write memory
```

> **Two absolutes.** Order: the proxy `permit` must precede the 80/443 `deny`, otherwise the deny catches the traffic to the proxy first. 
> 
> Final line: `permit ip any any` **mandatory**: without it, the implicit deny kills OSPF, mgmt, ICMP, everything. DNS over **TCP/53** is added alongside UDP/53 (responses > 512 B).

---

### <a id="étape-10--acl-dmz-restrict-deny-implicite--pas-de-permit-final"></a>Step 10: DMZ-RESTRICT ACL (implicit deny: NO final permit)

**Intent:** hostile zone: only PROXY egresses; WEB-PUBLIC can initiate nothing (anti reverse-shell).

```cisco
configure terminal

access-list DMZ-RESTRICT extended permit icmp any host 172.16.0.1
access-list DMZ-RESTRICT extended permit tcp host 172.16.0.20 any
access-list DMZ-RESTRICT extended permit udp host 172.16.0.20 any eq domain
access-list DMZ-RESTRICT extended permit tcp host 172.16.0.20 any eq domain
access-list DMZ-RESTRICT extended deny ip host 172.16.0.10 any
access-list DMZ-RESTRICT extended permit icmp host 172.16.0.20 any
access-group DMZ-RESTRICT in interface dmz

end
write memory
```

> The DMZ is hostile-by-default: the implicit deny **is** the protection, never end with `permit ip any any`. `deny ip host 172.16.0.10 any` (note `ip`, covers TCP/UDP/ICMP) stops WEB-PUBLIC from **initiating** anything; replies to legitimate inbound requests pass via the connection table. 
> 
> **Order:** `permit icmp any host 172.16.0.1` must precede the `deny host .10`, otherwise the ASA's self-ping to the web server is dropped. If you add them out of order, remove and re-add:
> ```cisco
> no access-list DMZ-RESTRICT extended deny ip host 172.16.0.10 any
> no access-list DMZ-RESTRICT extended permit icmp any host 172.16.0.1
> access-list DMZ-RESTRICT extended permit icmp any host 172.16.0.1
> access-list DMZ-RESTRICT extended deny ip host 172.16.0.10 any
> ```

---

### <a id="étape-11--durcissement--port-security-access--span-core"></a>Step 11: Hardening: port-security (Access) + SPAN (Core)

**Intent:** close the access layer (closes P2 #8) and place the passive IDS probe.

```cisco
! Each access switch, user ports Fa0/3 and Fa0/4

configure terminal

interface range fastEthernet 0/3 - 4
 switchport nonegotiate
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 
end
write memory
```

> `maximum 2`, not 1: a VoIP desk port sees two MACs (phone in Voice VLAN 30 + data PC).

```cisco
! CORE-SW: SPAN destination toward the IDS

configure terminal

interface GigabitEthernet1/0/5
 no shutdown
 switchport mode access
 switchport access vlan 99
 description IDS-SENSOR-SPAN-DEST
 exit
 
monitor session 1 source interface GigabitEthernet1/0/24
monitor session 1 destination interface GigabitEthernet1/0/5

end
write memory
```

> ⚠️ **PT limit (#23):** the `source` line is silently ignored: `show monitor session 1` shows only the destination. IDS = passive off-path; inline blocking is the IPS *simulated* by the OUTSIDE-IN denies.

**Validation:** on an access switch: `show port-security address` (2 `SecureSticky` MACs, V10 `Fa0/3` / V20 `Fa0/4`), `show port-security interface fa0/3` (`Secure-up`, `maximum 2`, `Restrict`). On CORE-SW: `show monitor session 1` (destination `Gi1/0/5`; source absent, as expected).

> 📷 **[P-11](#p-11)** port-security · **[P-12](#p-12)** SPAN/IDS.
# 3. Evidence & closure

## <a id="validation-de-bout-en-bout-gate-final"></a>End-to-end validation

The **group B protocol**: note the target line's hitcnt, run the flow once, re-read `show access-list`, confirm that **that precise line** incremented. A timeout has ten causes; a climbing counter has only one.

**✅ Group A: flows that must pass**

| #   | From  | Action                           | Expected                                        | Evidence                                                                              |
| --- | ----- | -------------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------- |
| A1  | PC1   | `ping 8.8.8.8`                   | 4/4 reply (TTL=251)                             | OUTSIDE-IN `echo-reply` hitcnt climbs: survives the ACL: [P-01](#p-01), [P-06](#p-06) |
| A2  | PC1   | `http://172.16.0.20`             | page (via proxy)                                | INSIDE line 1 permit hit: [P-02](#p-02)                                               |
| A4  | ASA   | `ping 172.16.0.10` / `.20`       | 5/5                                             | DMZ `permit icmp any host .1` hit: order OK: [P-03](#p-03), [P-06](#p-06)             |
| A5  | PROXY | `ping 8.8.8.8`                   | 4/4                                             | DMZ `permit icmp host .20` hit: [P-04](#p-04)                                         |
| A6  | PC1   | `ping 8.8.8.8` then `show xlate` | `ICMP PAT … flags i` dynamic + static `flags s` | the dynamic entry is the NAT proof: [P-05](#p-05)                                     |

> **A3 reclassified, blocked by design, not a failure.** PC1 → `http://172.16.0.10` (WEB-PUBLIC direct) times out: the reply is dropped by DMZ-RESTRICT `deny host .10`. Consistent with the forced proxy: internal hosts reach the web via `.20` (A2), never the DMZ server. A3 proves a second time that `deny host .10` works. [P-13](#p-13)

**⛔ Group B: flows that must fail (proven by counter)**

| #   | From         | Action               | Rule that must increment    | Evidence                                       |
| --- | ------------ | -------------------- | --------------------------- | ---------------------------------------------- |
| B1  | PC1          | `http://8.8.8.8`     | INSIDE `deny … eq www`      | hitcnt 24: [P-06](#p-06); visual [P-07](#p-07) |
| B2  | PC1          | `https://8.8.8.8`    | INSIDE `deny … eq 443`      | hitcnt 12: [P-08](#p-08); visual [P-07](#p-07) |
| B3  | PC-EXTERIEUR | `telnet 203.0.113.2` | OUTSIDE-IN `deny … eq 23`   | hitcnt 12: [P-08](#p-08); visual [P-09](#p-09) |
| B4  | PC-EXTERIEUR | `ping 203.0.113.2`   | OUTSIDE-IN `deny icmp echo` | hitcnt 9: [P-06](#p-06); visual [P-09](#p-09)  |
| B5  | WEB-PUBLIC   | `ping 8.8.8.8`       | DMZ `deny ip host .10`      | hitcnt 80: [P-06]v, [P-10](#p-10)              |

**🔒 Group C: hardening**

| # | Where | Command | Expected | Evidence |
|---|---|---|---|---|
| C1 | Access switch | `show port-security address` | 2 sticky MACs, `maximum 2`, `Secure-up` | V10 `Fa0/3` / V20 `Fa0/4`: [P-11](#p-11) |
| C2 | CORE-SW | `show monitor session 1` | dest `Gi1/0/5`; source absent (PT limit) | [P-12](#p-12) |

---

## <a id="dépannage-incidents-de-session"></a>Troubleshooting (session incidents)

### 3a: Build incidents

> Incidents encountered during the build, with the diagnostic that caught each one. Session history, **not** debts; each fixed the same day.

| # | Symptom | Cause | Diagnosis | Fix |
|---|---|---|---|---|
| 1 | PC1 `ping 8.8.8.8` → `Destination host unreachable` | DIST-SW1 without `0.0.0.0/0`; HQ showed `Gateway of last resort is not set` | `show ip route` on HQ + `show ip route ospf` on DIST | `ip route 0.0.0.0/0` **and** `default-information originate` on HQ: **the real blocker** |
| 2 | `show xlate` shows only the static; PC1 timeout | `inspect icmp` not applied yet → no ICMP xlate | `show run policy-map` | apply the global inspection policy (step 7) |
| 3 | Ping ASA→`8.8.8.8` = 0/5 despite route | control-plane ping from the ASA + the ASA ignores echo on outside: misleading test | `ping 203.0.113.2` from the ISP; L2 OK | test in the **flow direction** (PC1 → xlate), not device-to-device |
| 4 | `http://203.0.113.2` from PC-EXTERIEUR times out despite SYN arriving | Server-PT doesn't complete HTTP through an inbound static NAT in PT | OUTSIDE-IN line 7 hitcnt climbs while the browser times out | left as debt #22; mechanism proven by the counter |
| 5 | CDP empty ASA↔ISP, link up/up, 100 Mb/s on Gigabit | crossover auto-cabling + CDP off on the ASA: two red herrings; L2 healthy (ARP resolved) | `show arp` on the ISP + `show interface` counters | none: L2 healthy; the blocker was incident #1 |

> **Lesson:** a NAT'd flow is judged by `show xlate` and the ACL hit counter, **never** by a ping to/from an intermediate device.

### 3b: Reference commands (ASA 9.6 compatible)

```cisco
show running-config            ! zones, nameif, security-levels
show route                     ! ASA table + gateway of last resort
show nat                       ! NAT rules
show run object network        ! object subnets/hosts
show xlate                     ! active translations (the dynamic entry = the proof)
show access-list               ! ALL ACLs + counters
show running-config policy-map ! inspect icmp status
clear xlate                    ! flush the cache after a NAT change
show port-security address     ! sticky MACs
show monitor session 1         ! SPAN dest
show ip route ospf             ! DIST/Core: O*E2 0.0.0.0/0 = originated default
```

---

## <a id="registre-derreurs--dette-technique"></a>Error log & technical debt

> Final state of each item (closed / carried / deferred). Session troubleshooting is above. 
> ⚠️ **Cross-part numbering.** These numbers are identifiers referenced by other docs

| #   | Item                                              | Severity | Domain            | Status                                                            |
| --- | ------------------------------------------------- | -------- | ----------------- | ----------------------------------------------------------------- |
| 8   | Port Security absent on the access layer          | 🟠       | Security          | ✅ **Closed** (sticky, `maximum 2`, restrict)                      |
| 22  | Inbound HTTP rendering via static NAT fails in PT | 🟠       | Services          | 📋 PT debt: SYN proven (line 7 hitcnt); physical ASA only         |
| 23  | SPAN session source ignored                       | 🟠       | Detection         | 📋 PT debt: concept proven; physical Catalyst                     |
| 24  | ACL `log` / `time-exceeded` unsupported in PT     | 🟢       | Observability     | 📋 PT debt: `log` → SIEM (P8); PMTUD preserved via `unreachable`. |
| 25  | HTTPS proxying non-functional                     | 🟠       | Services          | 📋 PT debt: Squid `:3128` + `deny … eq 443` in prod               |
| 26  | `service tcp 80 80` unsupported                   | 🟢       | Services          | 📋 PT debt: static NAT + ACL `:80`                                |
| 15  | Core = single north-south L3 transit              | 🟠       | High availability | 📋 Carried debt                                                   |
| 9   | Voice VLAN 30 with no IP phone                    | 🟢       | Demonstrative     | 🔜 P5                                                             |
| L4  | 802.1X missing (basic port-security only)         | 🟠       | Security          | 🔜 P9 (RADIUS + 802.1X)                                           |


---

## <a id="annexe--captures-de-preuve"></a>Appendix: Evidence captures

**<a id="p-01"></a> [P-01] · A1 campus → Internet**: PC1 `ping 8.8.8.8` = 4/4, `TTL=251`

![Capture P3-12](../assets/captures/P3/Capture_P3_12.png)

**<a id="p-02"></a> [P-02] · A2 forced-proxy egress**: PC1 `http://172.16.0.20` = page served

![Capture P3-11](../assets/captures/P3/Capture_P3_11.png)

**<a id="p-03"></a> [P-03] · A4 ASA → DMZ**: ASA `ping 172.16.0.10` + `.20` = 5/5 each

![Capture P3-07](../assets/captures/P3/Capture_P3_07.png)

**<a id="p-04"></a> [P-04] · A5 proxy egress**: PROXY-SERVER `ping 8.8.8.8` = 4/4

![Capture P3-09](../assets/captures/P3/Capture_P3_09.png)

**<a id="p-05"></a> [P-05] · A6 NAT proof (the durable one)**: ASA `show xlate`: dynamic `ICMP PAT inside:192.168.10.50 → outside:203.0.113.2 flags i` + static `dmz:172.16.0.10 → 203.0.113.2 flags s`

![Capture P3-14](../assets/captures/P3/Capture_P3_14.png)

**<a id="p-06"></a> [P-06] · master counters**: ASA `show access-list`: OUTSIDE-IN `echo-reply`=16 / line 7 `www`=5 / `echo`=9; INSIDE `deny www`=24 / `permit ip`=8; DMZ `deny host .10`=80 / `permit icmp .1`=10

![Capture P3-06](../assets/captures/P3/Capture_P3_06.png)

**<a id="p-07"></a> [P-07] · B1+B2 direct web blocked (visual)**: PC1 `http://8.8.8.8:80 / :443` = Request Timeout (forced to proxy)

![Capture P3-10](../assets/captures/P3/Capture_P3_10.png)

**<a id="p-08"></a> [P-08] · B2+B3 counters**: ASA `show access-list`: INSIDE `deny 443`=12; OUTSIDE-IN `deny telnet`=12

![Capture P3-03](../assets/captures/P3/Capture_P3_03.png)

**<a id="p-09"></a> [P-09] · B3+B4 external attempts (visual)**: PC-EXTERIEUR `ping 203.0.113.2` = 100 % loss + `telnet 203.0.113.2` = Connection timed out

![Capture P3-04](../assets/captures/P3/Capture_P3_04.png)

**<a id="p-10"></a> [P-10] · B5 reverse-shell block**: WEB-PUBLIC `ping 8.8.8.8` = 100 % loss (cannot initiate egress)

![Capture P3-08](../assets/captures/P3/Capture_P3_08.png)

**<a id="p-11"></a> [P-11] · C1 port-security**: ACC-SW1 `show port-security address`: 2× `SecureSticky` (V10 `Fa0/3`, V20 `Fa0/4`), `maximum 2`, `Secure-up`, `Restrict`

![Capture P3-02](../assets/captures/P3/Capture_P3_02.png)

**<a id="p-12"></a> [P-12] · C2 SPAN / IDS**: CORE-SW `show monitor session 1`: destination `Gi1/0/5`; source silently absent (debt #23)

![Capture P3-01](../assets/captures/P3/Capture_P3_01.png)

**<a id="p-13"></a> [P-13] · A3 blocked-by-design**: PC1 `http://172.16.0.10` = Request Timeout: reply dropped by DMZ-RESTRICT `deny host .10` (not a fault)

![Capture P3-13](../assets/captures/P3/Capture_P3_13.png)

**<a id="p-14"></a> [P-14] · debt #22: inbound HTTP rendering**: PC-EXTERIEUR `http://203.0.113.2` = Request Timeout; the SYN reaches the server (OUTSIDE-IN line 7 hitcnt, [P-06](#p-06)), rendering blocked by PT. Works on a physical ASA

![Capture P3-05](../assets/captures/P3/Capture_P3_05.png)

---

⬅️ [Workflow P2](../P2/WORKFLOW.md) · ⬆️ [Contents](#sommaire) · [Part README](./README.md) · [Project overview](../README.md) · **Next: [Workflow P4](../P4/WORKFLOW.md)** – Spine-Leaf datacenter, Border Leafs, routed fabric, server tiers.
