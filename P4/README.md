# Partie 4 : Datacenter 

 **Concepts clés** : Spine-Leaf, Border Leafs, tiers serveurs & load balancer

- 💻**Outil** : Cisco Packet Tracer 9.0
- 🏷️ Plan d'adressage complet → [IPAM](../IPAM.md)
- 📝 Progression étape par étape → [WORKFLOW P4](./WORKFLOW.md)
- 🎓 **Certification :** CompTIA Network+

## Objectif

Construire le datacenter comme une fabric Spine-Leaf **routée**, boulée sur le campus **à travers le Core, pas le HQ-Router**. Le but n'est pas « faire pinguer les serveurs », c'est de prouver une forme de trafic précise : 

- **Est-Ouest** uniforme (`Leaf→Spine→Leaf`)
- **Nord-Sud** symétrique (`Leaf→Spine→BorderLeaf→Core→edge`), 
- Et une **application trois tiers** où les serveurs applicatifs sortent mais ne sont jamais joignables en entrée, chaque sens prouvé par un compteur.

## Contrainte structurante

- Les deux Border Leafs terminent sur le **Core** (`10.0.12.0/30`, `10.0.13.0/30`), pas un sur le Core et un sur le HQ-Router comme j'avais pu produire dans mon premier build. 
- Les deux chemins N-S sont de longueur identique → le Core apprend les sous-réseaux DC via BL1 **et** BL2 en **ECMP**, et le HQ-Router reste un pur edge campus.


## Décision de continuité (héritée de P3)

- Le campus atteint déjà Internet. 
- Le DC a seulement besoin qu'OSPF porte `172.16.2.0/24` + `172.16.3.0/24` jusqu'à l'edge, plus un objet PAT dédié. La route ASA + NAT se fait **en dernier**, une fois la fabric prouvée.

## Topologie logique

![Topologie P4](../assets/topologies/topology_p4.svg)

## Couverture CompTIA Network+

|Domaine|Concepts couverts|Statut|
|---|---|---|
|🗺️ Topologie & architecture|Fabric spine-leaf 2×2 + 2 border leafs · trafic Est-Ouest vs Nord-Sud · appli 3 tiers (présentation/app/data)|✅ E-O `Leaf→Spine→Leaf` · N-S via Border Leaf|
|🧭 Routage fabric|Accès routé (pas de VLAN étiré) · OSPF point-à-point sans DR/BDR · ECMP équi-coût · entre dans le résumé `/20` de P3|✅ voisins `FULL` · Core→DC via BL1 **et** BL2|
|🌐 Services|NAT/PAT pour le nouveau bloc interne|✅ objet PAT `DC-NET`|
|🛡️ Sécurité|Exposition en tiers (backend sortie-seule) · confinement ports inutilisés (VLAN 998)|✅ sortie ok / entrée refusée · 998 sur toute la fabric|
|🔁 Haute disponibilité|Concept load balancer / VIP|⚠️ documenté — LB fonctionnel = prod, hors PT|

## Conclusion

Une décision de topologie peut acheter une propriété que la config ne rattrapera jamais : les **deux Border Leafs sur le Core** rendent les chemins Nord-Sud symétriques (ECMP) et sortent le HQ-Router du datacenter. 

L'acquis de fond est la pensée « par sens de trafic » : Est-Ouest uniforme, Nord-Sud symétrique, tier applicatif qui **sort mais n'entre jamais**.

Chaque direction devant être prouvée séparément, parce qu'un flux qui marche dans un sens ne dit rien de l'autre.

---

⬆️ Progression étape par étape → [Workflow P4](./WORKFLOW.md) **Suivant : [Partie 5 — Téléphonie IP](../P5/README.md)** — CME co-localisé avec l'Active/root du VLAN 30, DHCP Option 150, SCCP, TFTP, frontière QoS. · [Vue d'ensemble du projet](../README.md)
