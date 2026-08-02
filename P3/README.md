# Partie 3 : DMZ & pare-feu

**Concepts clés** : ASA, DMZ, NAT/PAT & filtrage · **Certification :** CompTIA Network+  · **Outil :** Cisco Packet Tracer 9.0

- Plan d'adressage complet → [`IPAM.md`](../IPAM.md)
- Progression étape par étape → [`WORKFLOW P5`](./WORKFLOW.md)

---
## Objectif

Construire la frontière entre le réseau interne et Internet : 

- pare-feu ASA à trois zones, DMZ hébergeant les services exposés + un proxy, politique de sortie qui force le web interne par le proxy, 
- confinement reverse-shell sur le serveur publié, 
- sonde IDS passive. 

Le but n'est **pas** « faire passer du trafic » (c'était P2) — c'est de définir **ce qui a le droit de traverser, et dans quel sens**, et de prouver chaque règle par un **compteur**, pas par une capture.

**Contrainte structurante : l'ordre de build.** 

- Routage et NAT sont vérifiés sur un ASA **sans ACL** (les security-levels autorisent déjà inside→outside) *avant* toute ACL. 
- Déboguer un problème de routage à travers trois ACL à la fois est le gouffre de temps classique. 
- Les trois ACL portent trois philosophies opposées : le `permit ip any any` final est **obligatoire sur inside** et **interdit sur DMZ**.

**Décision de continuité (héritée de P2) :** 

- À la fin de P2, le campus n'atteignait Internet par personne. 
- P3 l'introduit via le lien HQ-Router → ASA inside. 
- Une route par défaut statique **ne suffit pas** : elle doit être poussée dans OSPF par `default-information originate`, même classe de piège « chemin de retour » que l'OFFER DHCP de P2.

---
## Topologie logique


![Topologie P3](../assets/topologies/topology_p3.svg)


---

## Couverture CompTIA Network+

| Domaine      | Concept                                                  | Statut                                             |
| ------------ | -------------------------------------------------------- | -------------------------------------------------- |
| 🛡️ Sécurité | Pare-feu 3 zones + security-levels                       | ✅ outside 0 / dmz 50 / inside 100                  |
| 🛡️ Sécurité | DMZ, trusted vs untrusted                                | ✅                                                  |
| 🛡️ Sécurité | ACL étendue + deny implicite                             | ✅ 3 ACL, 3 philosophies, prouvées par compteur     |
| 🛡️ Sécurité | Proxy de sortie forcé                                    | ✅ 80/443 direct deny, proxy permit d'abord         |
| 🛡️ Sécurité | Prévention reverse-shell                                 | ✅ `deny ip host 172.16.0.10 any` (hitcnt prouvé)   |
| 🛡️ Sécurité | Port-security (sticky, restrict)                         | ✅ clôt P2 #8                                       |
| 📡 Détection | IDS passif (SPAN / port mirroring)                       | ✅ dest prouvée ; source = limite PT (#23)          |
| 📡 Détection | IPS simulé (signatures ACL) · IDS vs IPS                 | ✅ deny 23/22/445 ; copie passive vs blocage inline |
| 🌐 Services  | NAT / PAT                                                | ✅ PAT dynamique ×2 + statique 1:1                  |
| 🌐 Services  | DNS TCP/53 (grosses réponses) · inspection ICMP stateful | ✅                                                  |
| 🧭 Routage   | Origination de la route par défaut dans OSPF             | ✅ `default-information originate`                  |
| 🧭 Routage   | LPM vs Administrative Distance                           | ✅ Null0 AD 254 flottant                            |
| 🧭 Routage   | Résumé de routes + verrou trou noir                      | ✅ résumés `/20` + `/16`, Null0 attrape             |

---
## Test & validation ✅

Cette section recense ce qui a été testé et validé : 

- Chaque `✅` cite un `[P-##]` de l'**[Annexe — Captures de preuve du WORKFLOW P3](https://claude.ai/chat/WORKFLOW.md#annexe--captures-de-preuve)**. 
- Un flux **bloqué** est prouvé par le **compteur de la ligne `deny`**, jamais par un timeout.

| **🧭 Routage & NAT**      | Résultat / compteur                                                                | Preuve                                          |
| ------------------------- | ---------------------------------------------------------------------------------- | ----------------------------------------------- |
| Route par défaut originée | `O*E2 0.0.0.0/0` sur DIST/Core — **inféré via le flux A1** (pas de capture dédiée) | [P-01](https://claude.ai/chat/WORKFLOW.md#p-01) |
| NAT dynamique + statique  | `show xlate` = `ICMP PAT … flags i` + `172.16.0.10 ↔ 203.0.113.2 flags s`          | [P-05](https://claude.ai/chat/WORKFLOW.md#p-05) |


| 🟢 Flux autorisés (permit) | Résultat / compteur                                        | Preuve                                                                                           |
| -------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| A1 · Campus → Internet     | PC1 `ping 8.8.8.8`, compteur OUTSIDE-IN `echo-reply` monte | [P-01](https://claude.ai/chat/WORKFLOW.md#p-01), [P-06](https://claude.ai/chat/WORKFLOW.md#p-06) |
| A2 / A5 · via proxy        | PC1 → proxy `.20` = page servie ; PROXY → `8.8.8.8` = 4/4  | [P-02](https://claude.ai/chat/WORKFLOW.md#p-02), [P-04](https://claude.ai/chat/WORKFLOW.md#p-04) |
| A4 · ASA → DMZ             | `ping .10`/`.20` = 5/5 (ordre DMZ OK)                      | [P-03](https://claude.ai/chat/WORKFLOW.md#p-03), [P-06](https://claude.ai/chat/WORKFLOW.md#p-06) |

> **A3 (PC1 → WEB-PUBLIC direct) est bloqué par conception** — [P-13](./WORKFLOW.md#p-13) : la réponse est jetée par DMZ-RESTRICT `deny host .10`. Cohérent avec le proxy forcé ; compté comme comportement voulu, pas comme un test échoué.

| ⛔ Flux bloqués (deny - compteur) | Résultat / compteur                               | Preuve                                                                                           |
| -------------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| B1 · HTTP direct                 | bloqué — INSIDE `deny eq www` hitcnt **24**       | [P-06](https://claude.ai/chat/WORKFLOW.md#p-06), [P-07](https://claude.ai/chat/WORKFLOW.md#p-07) |
| B2 · HTTPS direct                | bloqué — INSIDE `deny eq 443` hitcnt **12**       | [P-07](https://claude.ai/chat/WORKFLOW.md#p-07), [P-08](https://claude.ai/chat/WORKFLOW.md#p-08) |
| B3 · telnet externe → ASA        | bloqué — OUTSIDE-IN `deny eq 23` hitcnt **12**    | [P-08](https://claude.ai/chat/WORKFLOW.md#p-08), [P-09](https://claude.ai/chat/WORKFLOW.md#p-09) |
| B4 · ping externe → ASA          | bloqué — OUTSIDE-IN `deny icmp echo` hitcnt **9** | [P-06](https://claude.ai/chat/WORKFLOW.md#p-06), [P-09](https://claude.ai/chat/WORKFLOW.md#p-09) |
| B5 · WEB-PUBLIC → Internet       | bloqué — DMZ `deny ip host .10` hitcnt **80**     | [P-06](https://claude.ai/chat/WORKFLOW.md#p-06), [P-10](https://claude.ai/chat/WORKFLOW.md#p-10) |

| **🔒 Durcissement & supervision L2** | Résultat / compteur                                          | Preuve                                          |
| ------------------------------------ | ------------------------------------------------------------ | ----------------------------------------------- |
| Port-security                        | 2 MAC `SecureSticky` (V10 `Fa0/3`, V20 `Fa0/4`), `Secure-up` | [P-11](https://claude.ai/chat/WORKFLOW.md#p-11) |
| SPAN                                 | dest `Gi1/0/5`                                               | [P-12](https://claude.ai/chat/WORKFLOW.md#p-12) |

### ⚠️ Configuré / limité par PT

| Point                           | Limite                                                                      | Réf.                                                        |
| ------------------------------- | --------------------------------------------------------------------------- | ----------------------------------------------------------- |
| Rendu HTTP entrant → WEB-PUBLIC | SYN prouvé (OUTSIDE-IN ligne 7 hitcnt) ; **rendu applicatif** bloqué par PT | dette #22 · [P-14](https://claude.ai/chat/WORKFLOW.md#p-14) |
| Miroir SPAN source              | dest prouvée, source ignorée par PT                                         | dette #23 · [P-12](https://claude.ai/chat/WORKFLOW.md#p-12) |
| HTTPS via proxy                 | pas de `HTTP CONNECT` en PT                                                 | dette #25                                                   |

> Un `Request timed out` sur le premier paquet d'un flux frais (puis 0 %) est ARP + build de xlate, **pas** une faute.

---

## Conclusion

Le vrai apprentissage : l'ASA ne raisonne pas en ports mais en niveaux de confiance. Un changement de modèle mental qui ne s'intériorise pas dans la théorie, seulement en le vivant. 

Le reste en découle, à mes dépens : route par défaut injectée dans OSPF, `deny` ICMP nuancé pour préserver le PMTUD, agrégation verrouillée par Null0, règles prouvées au compteur de hits et non à la capture qui « a l'air de marcher ». 

Le moindre privilège est un scalpel, pas un mur où trop bloquer devient aussi une erreur de configuration.

---

⬆️ **Suivant : [Partie 4 — Datacenter Spine-Leaf](../P4/README.md)** — fabric routée, 2 Border Leafs sur le Core (ECMP N-S), tiers applicatif + stockage, VIP de load balancer. · [Vue d'ensemble du projet](../README.md)
