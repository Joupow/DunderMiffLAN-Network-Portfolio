# Partie 1 : Fondations LAN 3 niveaux

**Concepts clés** : Modèle hiérarchique Cisco 3 niveaux · VLANs · 802.1Q · STP · routage inter-VLAN · SVI · 

- 💻**Outil** : Cisco Packet Tracer 9.0
- 🏷️ Plan d'adressage complet → [IPAM.m](../IPAM.md)
- 📝 Progression étape par étape →[WORKFLOW P1](./WORKFLOW.md).
- 🎓 **Certification :** CompTIA Network+ 
## Objectif

Construire la fondation du LAN d'entreprise sur le modèle hiérarchique Cisco à trois niveaux comprenant : 

- Core, distributions et access
- Segmentation VLAN
- Plan de management dédié,
- Stabilisation STP
- Routage inter-VLAN *temporaire* sur le Core.

Tout ce qui suit : HSRP, OSPF, DMZ, datacenter, voix, Wi-Fi, dépend de la propreté de cette base.
## Contrainte structurante

- Le root STP est positionné dès cette partie sur la **Distribution**, aligné sur le plan HSRP de P2 (DIST-SW1 root `{10,30}`, DIST-SW2 root `{20,99}`). 
- Root L2 et passerelle L3 finiront ainsi sur le même switch par VLAN à partir de P2 : le principe *« le service suit l'Active »*, posé ici pour être hérité par toute la suite.

**Pourquoi une base temporaire sur le core ?** 

Une passerelle `.1` redondante sur deux switches de Distribution est **impossible sans FHRP** (même IP sur deux boîtiers = conflit). Livrer d'abord une base mono-boîtier qui fonctionne, puis la durcir avec HSRP + `/30` routé + OSPF en P2, de manière délibéré.
## Topologie logique

![Topologie P1](../assets/topologies/topology_p1.svg)

## Couverture CompTIA Network+

| Domaine                | Concepts couverts                                        | Statut                                               |
| ---------------------- | -------------------------------------------------------- | ---------------------------------------------------- |
| 🗺️ Topologie          | Hiérarchique 3 niveaux · star / collapsed core (comparé) | ✅ 7 switches, 3 niveaux                              |
| 🔌 Commutation         | VLANs · 802.1Q · VLAN natif durci · Rapid PVST+ · SVI    | ✅ 6 VLANs sur les 7 switches                         |
| 🔌 Commutation         | Voice VLAN · speed/duplex                                | ⚠️ démonstratif (poste en P5) · uplinks access 100M  |
| 🧭 Routage             | Inter-VLAN via SVI du Core                               | ✅ 4 SVIs · temporaire → durci en P2                  |
| 🏷️ Adressage IPv4     | Classe C `/24` · subnetting · hôtes statiques            | ✅ 4× /24 · 8 hôtes                                   |
| 🛡️ Sécurité           | Isolation de ports · durcissement bordure                | ✅ 998 + shutdown · BPDU Guard bordure only           |
| 🔁 Haute disponibilité | Dual-homing L2 · failover STP                            | ✅ mécanisme prouvé · ⏳ timing sous-seconde à mesurer |

Chaque point est prouvé par un **résultat, un appel ou un état** — jamais par un timeout. Validation de bout en bout, captures et incidents de build : **WORKFLOW — validation**.

## Bilan pédagogique 

Une base propre n'est pas une étape « facile » qu'on expédie : c'est la dette de toutes les parties suivantes. Ici, on pose les fondations avant les murs porteurs : la base L2 doit être stable avant d'empiler la redondance.

Deux incidents attrapés dès ce niveau : mismatch de VLAN natif, BPDU Guard sur un uplink, qui auraient contaminé HSRP, OSPF puis la voix s'ils étaient passés. 

L'apprentissage central reste le principe de séquençage : positionner le root STP _avant_ que la redondance existe, aligné sur le futur Active HSRP, parce qu'une passerelle `.1` redondée sur deux switches est **impossible sans FHRP**. 

---

⬆️ - Progression étape par étape →[Workflow P1](./WORKFLOW.md) **Suivant : [Partie 2 — Routage, redondance & services](../P2/README.md)** — migration des SVIs sur la Distribution, HSRP, OSPF point-à-point, DHCP centralisé + relais. · [Vue d'ensemble du projet](../README.md)
