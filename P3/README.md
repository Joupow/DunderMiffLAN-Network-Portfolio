# Part 3: DMZ & Firewall

**Key concepts**: ASA, DMZ, NAT/PAT & filtering

- 💻 **Tool**: Cisco Packet Tracer 9.0
- 🏷️ Full addressing plan → [IPAM](../IPAM.md)
- 📝 Step-by-step progression → [WORKFLOW P3](./WORKFLOW.md)
- 🎓 **Certification:** CompTIA Network+

## Logical topology


![P3 topology](../assets/topologies/topology_p3.svg)

## Objective

Build the border between the internal network and the Internet:

- a 3-zone ASA firewall, a DMZ hosting the exposed services + a proxy, an egress policy that forces internal web traffic through the proxy,
- reverse-shell containment on the published server,
- a passive IDS sensor.

The goal is **not** "make traffic flow" (that was P2); here it's to define **what is allowed to cross, and in which direction**, and to prove every rule with a **counter**, not a capture.

## Structural constraint: build order.

- Routing and NAT are verified on an ASA **with no ACLs** (the security-levels already permit inside→outside) *before* any ACL goes on.
- Debugging a routing problem through three ACLs at once is the classic time sink.
- The three ACLs carry three opposing philosophies: the trailing `permit ip any any` is **mandatory on inside** and **forbidden on the DMZ**.

## Continuity decision (inherited from P2)

- At the end of P2, the campus could reach the Internet through nobody.
- P3 introduces it via the HQ-Router → ASA inside link.
- A static default route **isn't enough**: it has to be pushed into OSPF with `default-information originate`, the same "return-path" trap class as P2's DHCP OFFER.

## CompTIA Network+ coverage

| Domain       | Concepts covered                                                                                                                                             | Status                            |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| 🛡️ Security | 3-zone firewall + security-levels · trusted/untrusted DMZ · extended ACLs + implicit deny · forced egress proxy · reverse-shell prevention · port-security | ✅ 3 zones (0/50/100) · 3 ACLs     |
| 📡 Detection | passive IDS (SPAN) · IPS simulated via ACL signatures · IDS vs IPS                                                                                          | ✅ · ⚠️ SPAN source limited by PT  |
| 🌐 Services  | NAT/PAT · DNS TCP/53 · stateful ICMP inspection                                                                                                             | ✅ PAT ×2 + static 1:1             |
| 🧭 Routing   | OSPF default route · LPM vs AD · summary + black-hole lock                                                                                                  | ✅ /20 + /16 summaries · Null0     |


## Conclusion

The real takeaway: the ASA doesn't reason in ports, it reasons in trust levels. That's a mental-model shift you can't internalize from theory, only by living it.

The rest follows from it, learned the hard way: default route injected into OSPF, a nuanced ICMP `deny` to preserve PMTUD, aggregation locked down with Null0, rules proven by hit counters rather than a capture that "looks like it works."

Least privilege is a scalpel, not a wall where over-blocking is just as much a misconfiguration as under-blocking.

---

⬅️ [Part 2: Routing & Redundancy](../P2/README.md) · ⬆️ [Project overview](../README.md) · 🔁 [Workflow P3](./WORKFLOW.md) · **Next: [Part 4: Datacenter](../P4/README.md)**: a routed fabric, 2 Border Leafs on the Core (N-S ECMP), an application tier + storage, a load-balancer VIP.
