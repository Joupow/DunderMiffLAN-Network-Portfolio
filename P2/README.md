# Partie 2 : Routage & redondance

 **Concepts clés** : Routage, HSRP, DHCP, OSFP P2P  ·  **Certification :** CompTIA Network+ · **Outil :** Cisco Packet Tracer 9.0
 

- Plan d'adressage complet → [`IPAM.md`](../IPAM.md)
- Progression étape par étape →[`WORKFLOW P2`](./WORKFLOW.md).

---
## Objectif

Transformer le LAN statique de P1 en un réseau routé, redondant et auto-adressé, autour de quatre chantiers :

- Migration des passerelles inter-VLAN du Core vers la Distribution en **HSRP dual-active** (VIP `.1`, physiques `.2`/`.3`)
- Uplinks Core↔Distribution convertis en **liens routés `/30`**, avec **OSPF** en point-à-point comme IGP du campus
- Adressage hôte centralisé : **autorité DHCP unique** sur le HQ-Router + relais `ip helper-address`
- **Durcissement L2** des ports d'accès et **équilibrage STP PVST+** aligné sur les rôles HSRP

Cela solde les dettes critiques laissées ouvertes par P1 : SVIs sur le Core, SPOF inter-VLAN, passerelles sans redondance. 

Tout ce qui suit, DMZ, datacenter, voix, Wi-Fi s'appuie sur ce socle routé et redondant.

**Contrainte structurante :** 

- La répartition HSRP place les deux VLANs lourds sur des boîtiers différents : DIST-SW1 Active `{10,30}`, DIST-SW2 Active `{20,99}`. 
- Pour chaque VLAN : **Active HSRP = root STP = service hébergé** — *« le service suit l'Active »*.

**Décision de continuité (héritée de P1) :** 

- Le root STP a été posé en P1 sur la Distribution selon ce split
- P2 aligne HSRP dessus, sans re-toucher STP. 
- La bascule est ordonnée **Core d'abord** : router les uplinks du Core et retirer ses SVIs data *avant* de lever les VIP de la Distribution, pour que la passerelle `.1` ne soit jamais revendiquée par deux boîtiers à la fois.


---
## Topologie logique

![Topologie P2](../assets/topologies/topology_p2.svg)

---
## Couverture CompTIA Network+

| Domaine                | Concept                                    | Statut                                           |
| ---------------------- | ------------------------------------------ | ------------------------------------------------ |
| 🧭 Routage             | OSPFv2 mono-aire (aire 0)                  | ✅ tous les voisins `FULL`                        |
| 🧭 Routage             | Type OSPF point-à-point                    | ✅ chaque `/30`, **zéro DR/BDR**                  |
| 🧭 Routage             | Router-ID (manuel)                         | ✅ codé en dur + `clear ip ospf process`          |
| 🧭 Routage             | `passive-interface` sélectif               | ✅ default + un-passive sur le transit uniquement |
| 🧭 Routage             | Ports routés (`no switchport`)             | ✅ uplinks Core + DIST                            |
| 🧭 Routage             | ECMP                                       | ✅ le Core atteint les VLANs via les deux DIST    |
| 🔁 Haute disponibilité | FHRP — HSRP (VIP / priorité / preempt)     | ✅ failover **et** preempt prouvés, deux sens     |
| 🔁 Haute disponibilité | Répartition Active/Standby HSRP            | ✅ DIST1 `{10,30}` · DIST2 `{20,99}`              |
| 🔌 Commutation         | Root STP aligné sur l'Active HSRP          | ✅ les quatre VLANs vérifiés                      |
| 🌐 Services            | Serveur DHCP (scopes, exclusions, options) | ✅ VLAN 10 & 20 sur HQ-Router                     |
| 🌐 Services            | Relais DHCP (`ip helper-address`)          | ✅ chemin unique par VLAN                         |
| 🏷️ Adressage IP       | Subnetting transit `/30`                   | ✅ 3 liens point-à-point, sans chevauchement      |

---

## Test & validation ✅

Cette section recense ce qui a été testé et validé : 

- Chaque `✅` cite un `[P-##]` de l'**[Annexe — Captures de preuve du WORKFLOW P2](./WORKFLOW.md#annexe--captures-de-preuve)**.
- Chaque flux (routage, bascule, ping) est prouvé par un **résultat, un appel ou un état**, pas par un timeout.

| **🧭 Routage (OSPF / underlay)**        | Résultat / état                                                        | Preuve                                                                                                                                                                                             |
| --------------------------------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Uplinks routés du Core                  | `up` sur `Gi1/0/1`, `/2`, `/24`                                        | [P-01](https://claude.ai/chat/WORKFLOW.md#p-01)                                                                                                                                                    |
| Adjacences OSPF `FULL/ -`, **0 DR/BDR** | Core voit 3 voisins ; DIST1, DIST2, HQ confirmés                       | [P-02](https://claude.ai/chat/WORKFLOW.md#p-02), [P-04](https://claude.ai/chat/WORKFLOW.md#p-04), [P-06](https://claude.ai/chat/WORKFLOW.md#p-06), [P-08](https://claude.ai/chat/WORKFLOW.md#p-08) |
| Table de routage OSPF du Core           | ECMP vers les VLANs via les deux `/30` ; routes DIST1 + propagation HQ | [P-03](https://claude.ai/chat/WORKFLOW.md#p-03), [P-05](https://claude.ai/chat/WORKFLOW.md#p-05), [P-07](https://claude.ai/chat/WORKFLOW.md#p-07)                                                  |

| **🔁 Haute disponibilité (HSRP)** | Résultat / état                                                       | Preuve                                                                                                                                                                                             |
| --------------------------------- | --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Répartition HSRP                  | DIST1 Active `{10,30}`, DIST2 Active `{20,99}`, preempt flaggé        | [P-09](https://claude.ai/chat/WORKFLOW.md#p-09), [P-10](https://claude.ai/chat/WORKFLOW.md#p-10)                                                                                                   |
| Cohérence STP ↔ HSRP              | Root STP = l'Active du VLAN (`{10}`, `{20}`, `{30}`, `{99}`)          | [P-11](https://claude.ai/chat/WORKFLOW.md#p-11), [P-12](https://claude.ai/chat/WORKFLOW.md#p-12), [P-13](https://claude.ai/chat/WORKFLOW.md#p-13), [P-14](https://claude.ai/chat/WORKFLOW.md#p-14) |
| **Failover**                      | DIST1 SVI 10 coupé → DIST2 promu Active, ping rétabli (**~3 perdus**) | [P-15](https://claude.ai/chat/WORKFLOW.md#p-15), [P-16](https://claude.ai/chat/WORKFLOW.md#p-16), [P-17](https://claude.ai/chat/WORKFLOW.md#p-17)                                                  |
| **Preempt**                       | DIST1 SVI 10 `no shutdown` → priorité 110 reprend l'Active            | [P-18](https://claude.ai/chat/WORKFLOW.md#p-18), [P-19](https://claude.ai/chat/WORKFLOW.md#p-19)                                                                                                   |

| **🌐 Services (DHCP)** | Résultat / état                             | Preuve                                          |
| ---------------------- | ------------------------------------------- | ----------------------------------------------- |
| Baux DHCP distribués   | `.10.50–.53`, `.20.51–.54` (serveur unique) | [P-20](https://claude.ai/chat/WORKFLOW.md#p-20) |

| **📦 Connectivité** | Résultat / état                             | Preuve                                          |
| ------------------- | ------------------------------------------- | ----------------------------------------------- |
| Ping inter-VLAN     | PC V10 → `192.168.20.51`, TTL=127 (un saut) | [P-21](https://claude.ai/chat/WORKFLOW.md#p-21) |

### ⚠️ Configuré / limité par PT

| Point                                         | Limite                                                             | Réf.      |
| --------------------------------------------- | ------------------------------------------------------------------ | --------- |
| Relais DHCP pendant un failover               | comportement non capturé — pas de redondance DHCP (serveur unique) | dette #16 |
| Joignabilité management depuis **chaque** ACC | vérifiée par sondage, non exhaustive                               | —         |


> Le timeout sur le premier paquet d'un flux inter-VLAN frais (puis 0 %) est ARP + convergence, **pas** une faute.

---

## Conclusion

La leçon la plus transférable n'est pas « configurer HSRP », c'est l'**ordre de bascule** : router les uplinks du Core et retirer ses SVIs data _avant_ de lever les VIP de la Distribution, sous peine de voir la `.1` revendiquée par deux boîtiers (split-brain). 

Deuxième acquis, contre-intuitif : la plupart des pannes de connectivité sont des problèmes de **chemin retour**, pas d'aller — le relais DHCP ne casse pas à la requête mais à l'OFFER qui ne sait pas revenir. On ne le comprend qu'en le débuggant.

Ajouter de la haute disponibilité ne supprime pas le risque : ça l'échange contre un risque plus discret. HSRP et OSPF ont apporté la redondance mais introduit des modes de défaillance silencieux — une collision de Router-ID, VLAN 99 qui fuit dans le plan de contrôle — dont aucun ne lève d'erreur évidente. La leçon : chaque composant ajouté pour la résilience est aussi une nouvelle chose qui peut casser en silence ; la redondance n'est réelle qu'une fois le chemin de panne vérifié, pas seulement le chemin nominal.

---

⬆️ **Suivant : [Partie 3 — DMZ & pare-feu](../P3/README.md)** — ASA 3 zones, NAT/PAT, 3 ACL, IDS/SPAN, origination de la route par défaut + verrou résumé/Null0. · [Vue d'ensemble du projet](../README.md)
