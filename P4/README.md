# Partie 4 — Datacenter : Spine-Leaf, Border Leafs, tiers serveurs & load balancer

**Bloc :** Datacenter TheBigOffice · **Outil :** Cisco Packet Tracer (Catalyst 3650) · **Certification :** CompTIA Network+

> **Partie 4 · Spine-Leaf routée**
> 
> **Statut :** ✅ Validé 
> 
> Fabric Spine-Leaf routée 2×2, **2 Border Leafs sur le Core** (symétrie N-S, HQ-Router hors du chemin DC), tiers applicatif + data joignables, **E-O et N-S prouvés par résultat**, **sortie Internet prouvée par compteur NAT**, **isolation DMZ→APP prouvée par compteur d'ACL**, ports inutilisés en VLAN 998 
> 
> Plan d'adressage complet → [`IPAM.md`](../IPAM.md).
> Progression étape par étape → [`WORKFLOW P4`](./WORKFLOW.md)


---
## Objectif

Construire le datacenter comme une fabric Spine-Leaf **routée**, boulée sur le campus **à travers le Core, pas le HQ-Router**. Le but n'est pas « faire pinguer les serveurs » — c'est de prouver une forme de trafic précise : **Est-Ouest** uniforme (`Leaf→Spine→Leaf`), **Nord-Sud** symétrique (`Leaf→Spine→BorderLeaf→Core→edge`), et une **application trois tiers** où les serveurs applicatifs sortent mais ne sont jamais joignables en entrée — chaque sens prouvé par un compteur.

**Contrainte structurante.** Les deux Border Leafs terminent sur le **Core** (`10.0.12.0/30`, `10.0.13.0/30`), pas un sur le Core et un sur le HQ-Router. Les deux chemins N-S sont de longueur identique → le Core apprend les sous-réseaux DC via BL1 **et** BL2 en **ECMP**, et le HQ-Router reste un pur edge campus.

**Décision de continuité (héritée de P3).** Le campus atteint déjà Internet. Le DC a seulement besoin qu'OSPF porte `172.16.2.0/24` + `172.16.3.0/24` jusqu'à l'edge, plus un objet PAT dédié. La route ASA + NAT se fait **en dernier**, une fois la fabric prouvée.

![Topologie P4](../assets/topologies/topology_p4.svg)

---
## Couverture CompTIA Network+

| Domaine             | Concept                                         | Statut                                       |
| ------------------- | ----------------------------------------------- | -------------------------------------------- |
| Architecture        | Fabric Spine-Leaf (leaf-spine)                  | ✅ 2×2 + 2 border leafs                       |
| Architecture        | Trafic Est-Ouest vs Nord-Sud                    | ✅ E-O `Leaf→Spine→Leaf`, N-S via Border Leaf |
| Architecture        | Application trois tiers (présentation/app/data) | ✅ DMZ / VLAN 210 / VLAN 220                  |
| Routage             | Accès routé (pas de VLAN étiré dans la fabric)  | ✅ chaque port fabric `no switchport` + `/30` |
| Routage             | OSPF point-à-point (pas de DR/BDR)              | ✅ tous les voisins `FULL/ -`                 |
| Routage             | ECMP / partage de charge équi-coût              | ✅ Core → DC via BL1 **et** BL2               |
| Routage             | Réutilisation du résumé (P3 `/20`)              | ✅ la fabric entre dans le résumé existant    |
| Services            | NAT/PAT pour un nouveau bloc interne            | ✅ objet PAT `DC-NET`                         |
| Sécurité            | Exposition en tiers (backend sortie-seule)      | ✅ sortie prouvée, entrée refusée par hitcnt  |
| Sécurité            | Confinement des ports inutilisés (VLAN 998)     | ✅ sur tous les switches de fabric            |
| Haute disponibilité | Concept load balancer / VIP                     | ✅ documenté (LB fonctionnel = prod)          |

---

## Matrice de validation (locale)


> Chaque `✅` cite un `[P-##]` de l'**[Annexe — Captures de preuve du WORKFLOW P4](./WORKFLOW.md)**. Chaque flux (sortie, isolation) est prouvé par un **compteur**, pas par un timeout.

| ✅ Prouvé (par résultat ou compteur)                                                                                                 | ⚠️ Configuré / limité par PT                                                                                            |
| ----------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Adjacences OSPF **toutes `FULL/ -`, zéro DR/BDR** : Spine ×4 [P-01], BL ×3 [P-03], Leaf ×2 [P-04], Core ×5 [P-05]                   | Round-robin LB : `http://172.16.2.10` sert toujours la page du LB, jamais `.11`/`.12` (pas de moteur LB — dette) [P-16] |
| ECMP N-S : Core `route 172.16.2.0` via `10.0.12.1` (BL1) **et** `10.0.13.1` (BL2) [P-06]                                            | SAN block-store : seule une réponse ping/HTTP (rôle conceptuel — dette)                                                 |
| Propagation DC vers l'edge : HQ-Router `O 172.16.2.0` + `O 172.16.3.0` [P-07] ; ASA `show route` = `3 subnets` [P-20]               | DMZ→APP : **intentionnellement bloqué** (P3 `DMZ-RESTRICT` ligne 5, hitcnt=4) jusqu'à l'ajout du permit P3 [P-17]       |
| E-O : APP-WEB1 → SAN `172.16.3.10` = 4/4 (2e run) [P-10]                                                                            |                                                                                                                         |
| N-S : PC campus → APP-WEB1 = 4/4 ; `tracert` = `DIST→Core(BL1)→Spine→Leaf1` [P-11]                                                  |                                                                                                                         |
| Sortie Internet : APP-WEB1 `ping 8.8.8.8` = `TTL=249` [P-13] ; ASA `DC-NET translate_hits=3` (le **compteur** est la preuve) [P-12] |                                                                                                                         |
| Services : `http://.11` = « Web-App 1 » [P-14], `http://.12` = « Web-App 2 » (distinct) [P-15]                                      |                                                                                                                         |
| Isolation : WEB-PUBLIC → APP bloqué — `DMZ-RESTRICT` deny hitcnt=4 (paquet mort à l'ACL P3 — fabric innocente) [P-17]               |                                                                                                                         |
| Durcissement : ports fabric inutilisés `disabled` VLAN 998 [P-09]                                                                   |                                                                                                                         |

> Un `Request timed out` en tête (jusqu'à 3 sur les longs flux N-S/sortie) sur le **premier** paquet est un build ARP + xlate/CEF, **pas** une faute. Les ports campus qui flashent ambre puis vert (0 % de perte) sont une reconvergence STP cosmétique, artefact PT.

---

## Registre d'erreurs & dette technique

> Registre complet en état final (dettes PT LB/SAN/Jumbo, dépendance P3 DMZ→APP, décisions locales, `#15` porté) et dépannage de session — **source unique : [WORKFLOW P4 §5](./WORKFLOW.md)**. Non dupliqué ici.


<!-- TODO conclusion P4 : 2-3 lignes sur l'apprentissage clé (ECMP N-S symétrique, Border Leafs sur le Core, résumé /20 réutilisé sans renumérotation). À rédiger avant publication ou supprimer ce commentaire. -->

---

⬆️ **Suivant : [Partie 5 — Téléphonie IP](../P5/README.md)** — CME co-localisé avec l'Active/root du VLAN 30, DHCP Option 150, SCCP, TFTP, frontière QoS. · [Vue d'ensemble du projet](../README.md)
