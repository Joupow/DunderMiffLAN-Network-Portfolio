# Partie 3 — Périmètre : ASA, DMZ, NAT/PAT & filtrage

**Bloc :** Périmètre TheBigOffice · **Outil :** Cisco Packet Tracer (ASA 9.6) · **Certification :** CompTIA Network+

> **Partie 3 · pare-feu ASA & DMZ**
> 
> 3 zones : outside `203.0.113.2` (SL 0) / dmz `172.16.0.1` (SL 50) / inside `192.168.200.1` (SL 100) · route par défaut originée dans OSPF · résumés internes verrouillés par Null0 · NAT/PAT + publication statique · ACL OUTSIDE-IN / INSIDE-FORCED-PROXY / DMZ-RESTRICT · WEB-PUBLIC `.10`, PROXY `.20` · IDS passif (SPAN) · port-security sticky.
> 
> Plan d'adressage complet → [`IPAM.md`](../IPAM.md).

**Statut :** ✅ Validé — edge ASA 3 zones, **le campus atteint Internet** (origination du défaut prouvée), NAT/PAT fonctionnel, **3 ACL aux philosophies opposées prouvées par compteur de hits**, proxy forcé + prévention reverse-shell, IDS passif (SPAN), port-security (clôt P2 #8) · 5 incidents corrigés · 0 déviation · 4 limitations PT (mécanisme prouvé pour chacune)

---

## Objectif

Construire la frontière entre le réseau interne et Internet : pare-feu ASA à trois zones, DMZ avec services exposés + proxy, sortie web forcée par le proxy, confinement reverse-shell, sonde IDS passive. Le but n'est **pas** « faire passer du trafic » (c'était P2) — c'est de définir **ce qui a le droit de traverser, et dans quel sens**, et de prouver chaque règle par un **compteur**, jamais par un timeout.

**Contrainte structurante — les trois philosophies opposées.** Trois ACL, trois réflexes contraires. Le `permit ip any any` final est **obligatoire** sur inside (sinon le deny implicite tue OSPF et le mgmt) et **interdit** sur DMZ (le deny implicite *est* la protection).

| ACL | Interface | Philosophie | Se termine par |
|---|---|---|---|
| OUTSIDE-IN | outside (in) | deny par défaut ; n'ouvre que HTTP→WEB-PUBLIC ; ICMP chirurgical | (deny implicite) |
| INSIDE-FORCED-PROXY | inside (in) | permit proxy d'abord, puis **deny** 80/443 direct, DNS autorisé | **`permit ip any any` — OBLIGATOIRE** |
| DMZ-RESTRICT | dmz (in) | zone hostile ; seul PROXY sort ; WEB-PUBLIC `deny ip host` (anti reverse-shell) | (deny implicite) — **JAMAIS `permit ip any any`** |

**Décision de continuité (héritée de P2).** À la fin de P2, le campus n'atteignait Internet par personne. P3 l'introduit via HQ-Router → ASA inside. Une route par défaut statique **ne suffit pas** : `default-information originate` la pousse dans OSPF — même piège « chemin de retour » que l'OFFER DHCP de P2.

![Topologie P3](../assets/topologies/topology_p3.svg)

---

## Niveaux & équipements

| Rôle | Équipement | Rôle dans la partie |
|---|---|---|
| Pare-feu edge | **ASA-EDGE** (ASA 5506-X) — *nouveau* | 3 zones (0/50/100) ; NAT/PAT ; 3 ACL ; inspection ICMP stateful |
| Internet | **ISP-Router** (2911) — *nouveau* | Internet simulé : loopback `8.8.8.8` ; outside `/30` ; PC de test externe |
| Services (routage) | **HQ-Router** (ISR 2911) | `/30` ASA-inside, route par défaut, **origine `0.0.0.0/0` dans OSPF** ; verrous Null0 |
| Switch DMZ | **DMZ-SW** (2960) — *nouveau* | L2 pour les deux serveurs DMZ |
| Serveurs DMZ | **WEB-PUBLIC** `.10` / **PROXY** `.20` — *nouveaux* | Front publié / sortie proxy forcé |
| Détection | **IDS-Sensor** (`.99.20`) — *nouveau* | Destination SPAN de l'uplink edge du Core (passif) |
| Access | 4× Catalyst **2960** | **Port-security fermée** — sticky, `maximum 2`, `violation restrict` |

Tout le campus P1/P2 est **inchangé** — P3 boule le périmètre sur le HQ-Router existant (`Gi0/0`=Core intouché ; l'ASA arrive sur `Gi0/1`).

---

## Couverture CompTIA Network+

| Domaine | Concept | Statut |
|---|---|---|
| Sécurité | Pare-feu 3 zones + security-levels | ✅ outside 0 / dmz 50 / inside 100 |
| Sécurité | DMZ, trusted vs untrusted | ✅ |
| Sécurité | ACL étendue + deny implicite | ✅ 3 ACL, 3 philosophies, prouvées par compteur |
| Sécurité | Proxy de sortie forcé | ✅ 80/443 direct deny, proxy permit d'abord |
| Sécurité | Prévention reverse-shell | ✅ `deny ip host 172.16.0.10 any` (hitcnt prouvé) |
| Sécurité | Port-security (sticky, restrict) | ✅ clôt P2 #8 |
| Détection | IDS passif (SPAN / port mirroring) | ✅ dest prouvée ; source = limite PT (#23) |
| Détection | IPS simulé (signatures ACL) · IDS vs IPS | ✅ deny 23/22/445 ; copie passive vs blocage inline |
| Services | NAT / PAT | ✅ PAT dynamique ×2 + statique 1:1 |
| Services | DNS TCP/53 (grosses réponses) · inspection ICMP stateful | ✅ |
| Routage | Origination de la route par défaut dans OSPF | ✅ `default-information originate` |
| Routage | LPM vs Administrative Distance | ✅ Null0 AD 254 flottant |
| Routage | Résumé de routes + verrou trou noir | ✅ résumés `/20` + `/16`, Null0 attrape |

---

## Matrice de validation (locale)

> Chaque `✅` cite un `[P-##]` de l'**[Annexe — Captures de preuve du WORKFLOW P3](./WORKFLOW.md#annexe--captures-de-preuve)**. Un flux **bloqué** est prouvé par le **compteur de la ligne deny**, jamais par un timeout.

| ✅ Prouvé (par résultat ou compteur) | ⚠️ Configuré / limité par PT |
|---|---|
| Campus → Internet : PC1 `ping 8.8.8.8`, compteur OUTSIDE-IN `echo-reply` monte — [P-01](./WORKFLOW.md#p-01), [P-06](./WORKFLOW.md#p-06) | Rendu HTTP entrant vers WEB-PUBLIC : SYN prouvé (OUTSIDE-IN ligne 7 hitcnt), rendu bloqué par PT — dette #22 — [P-14](./WORKFLOW.md#p-14) |
| Route par défaut originée (`O*E2 0.0.0.0/0` sur DIST/Core) — prouvée par le flux A1, pas de capture dédiée — [P-01](./WORKFLOW.md#p-01) | Miroir source SPAN : dest prouvée, source ignorée par PT — dette #23 — [P-12](./WORKFLOW.md#p-12) |
| NAT dynamique + statique : `show xlate` = `ICMP PAT … flags i` + `172.16.0.10 ↔ 203.0.113.2 flags s` — [P-05](./WORKFLOW.md#p-05) | HTTPS via proxy : pas de HTTP CONNECT en PT — dette #25 |
| A2 PC1 → proxy `.20` = page servie — [P-02](./WORKFLOW.md#p-02) · A5 PROXY → `8.8.8.8` = 4/4 — [P-04](./WORKFLOW.md#p-04) | |
| A4 ASA → DMZ `ping .10`/`.20` = 5/5 (ordre DMZ OK) — [P-03](./WORKFLOW.md#p-03), [P-06](./WORKFLOW.md#p-06) | |
| **B1** PC1 HTTP direct bloqué — INSIDE `deny eq www` hitcnt 24 — [P-06](./WORKFLOW.md#p-06), [P-07](./WORKFLOW.md#p-07) | |
| **B2** PC1 HTTPS direct bloqué — INSIDE `deny eq 443` hitcnt 12 — [P-07](./WORKFLOW.md#p-07), [P-08](./WORKFLOW.md#p-08) | |
| **B3** telnet externe → ASA bloqué — OUTSIDE-IN `deny eq 23` hitcnt 12 — [P-08](./WORKFLOW.md#p-08), [P-09](./WORKFLOW.md#p-09) | |
| **B4** ping externe → ASA bloqué — OUTSIDE-IN `deny icmp echo` hitcnt 9 — [P-06](./WORKFLOW.md#p-06), [P-09](./WORKFLOW.md#p-09) | |
| **B5** WEB-PUBLIC → Internet bloqué — DMZ `deny ip host .10` hitcnt 80 — [P-06](./WORKFLOW.md#p-06), [P-10](./WORKFLOW.md#p-10) | |
| Port-security : 2 MAC `SecureSticky` (V10 `Fa0/3`, V20 `Fa0/4`), `Secure-up` — [P-11](./WORKFLOW.md#p-11) · SPAN dest `Gi1/0/5` — [P-12](./WORKFLOW.md#p-12) | |

> **A3 (PC1 → WEB-PUBLIC direct) est bloqué par conception** — [P-13](./WORKFLOW.md#p-13) : la réponse est jetée par DMZ-RESTRICT `deny host .10`. Cohérent avec le proxy forcé ; compté comme comportement voulu, pas comme un test échoué.
> Un `Request timed out` sur le premier paquet d'un flux frais (puis 0 %) est ARP + build de xlate, **pas** une faute.

---

## Registre d'erreurs & dette technique

> Registre complet en état final (dettes PT `#22/#23/#24/#25/#26`, portées, différées), dépannage de session, et clôture de la dette P2 #8 — **source unique : [WORKFLOW P3 §5](./WORKFLOW.md)**. Non dupliqué ici.

---

⬆️ **Suivant : [Partie 4 — Datacenter Spine-Leaf](../P4/README.md)** — fabric routée, 2 Border Leafs sur le Core (ECMP N-S), tiers applicatif + stockage, VIP de load balancer. · [Vue d'ensemble du projet](../README.md)
