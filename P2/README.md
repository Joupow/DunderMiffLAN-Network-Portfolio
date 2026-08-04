# Partie 2 : Routage & redondance

 **Concepts clés** : Routage, HSRP, DHCP, OSFP P2P
 
- 💻**Outil** : Cisco Packet Tracer 9.0
- 🏷️ Plan d'adressage complet → [IPAM](../IPAM.md)
- 📝 Progression étape par étape → [WORKFLOW P2](./WORKFLOW.md)
- 🎓 **Certification :** CompTIA Network+ 

## Topologie logique

![Topologie P2](../assets/topologies/topology_p2.svg)
## Objectif

Transformer le LAN statique de P1 en un réseau routé, redondant et auto-adressé, autour de quatre chantiers :

- Migration des passerelles inter-VLAN du Core vers la Distribution en **HSRP dual-active** (VIP `.1`, physiques `.2`/`.3`)
- Uplinks Core↔Distribution convertis en **liens routés `/30`**, avec **OSPF** en point-à-point comme IGP du campus
- Adressage hôte centralisé : **autorité DHCP unique** sur le HQ-Router + relais `ip helper-address`
- **Durcissement L2** des ports d'accès et **équilibrage STP PVST+** aligné sur les rôles HSRP

Cela solde les dettes critiques laissées ouvertes par P1 : SVIs sur le Core, SPOF inter-VLAN, passerelles sans redondance. 

Tout ce qui suit, DMZ, datacenter, voix, Wi-Fi s'appuie sur ce socle routé et redondant.

## Contrainte structurante 

- La répartition HSRP place les deux VLANs lourds sur des boîtiers différents : DIST-SW1 Active `{10,30}`, DIST-SW2 Active `{20,99}`. 
- Pour chaque VLAN : **Active HSRP = root STP = service hébergé** — *« le service suit l'Active »*.

## Décision de continuité (héritée de P1) 

- Le root STP a été posé en P1 sur la Distribution selon ce split
- P2 aligne HSRP dessus, sans re-toucher STP. 
- La bascule est ordonnée **Core d'abord** : router les uplinks du Core et retirer ses SVIs data *avant* de lever les VIP de la Distribution, pour que la passerelle `.1` ne soit jamais revendiquée par deux boîtiers à la fois.

## Couverture CompTIA Network+

| Domaine                | Concepts couverts                                                                                                       | Statut                                                   |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| 🧭 Routage OSPF        | point-à-point sans DR/BDR · Mono-aire (aire 0)  · Router-ID manuel · `passive-interface` sélectif · ports routés · ECMP | ✅ tous voisins `FULL`                                    |
| 🔁 Haute disponibilité | HSRP (VIP / priorité / preempt) · répartition Active/Standby                                                            | ✅ DIST1 `{10,30}` · DIST2 `{20,99}` · failover deux sens |
| 🔌 Commutation         | **Root STP aligné sur l'Active HSRP** : _le service suit l'Active_                                                      | ✅ 4 VLANs                                                |
| 🌐 Services            | DHCP (scopes, exclusions, options) · relais `ip helper-address`                                                         | ✅ VLAN 10 & 20 sur HQ-Router                             |
| 🏷️ Adressage          | Subnetting transit `/30`                                                                                                | ✅ 3 liens P2P, sans chevauchement                        |

## Conclusion

La leçon la plus transférable n'est pas « configurer HSRP », c'est l'**ordre de bascule** : router les uplinks du Core et retirer ses SVIs data _avant_ de lever les VIP de la Distribution, sous peine de voir la `.1` revendiquée par deux boîtiers (split-brain). 

Deuxième acquis, contre-intuitif : la plupart des pannes de connectivité sont des problèmes de **chemin retour**, pas d'aller — le relais DHCP ne casse pas à la requête mais à l'OFFER qui ne sait pas revenir. On ne le comprend qu'en le débuggant.

La leçon : chaque composant ajouté pour la résilience est aussi une nouvelle chose qui peut casser en silence ; la redondance n'est réelle qu'une fois le chemin de panne vérifié, pas seulement le chemin nominal.

---

Progression étape par étape → [Workflow P2](./WORKFLOW.md) **Suivant : [Partie 3 : DMZ & pare-feu](../P3/README.md)** - ASA 3 zones, NAT/PAT, 3 ACL, IDS/SPAN, origination de la route par défaut + verrou résumé/Null0. · [Vue d'ensemble du projet](../README.md)
