# Partie 5 : VoIP 

 **Concepts clés** : VoIP · CME · TFTP · DHCP Option 150 · QoS voix

- 💻**Outil** : Cisco Packet Tracer 9.0
- 🏷️ Plan d'adressage complet → [IPAM](../IPAM.md)
- 📝 Progression étape par étape → [WORKFLOW P5](./WORKFLOW.md)
- 🎓 **Certification :** CompTIA Network+

## Topologie logique

![Topologie P5](../assets/topologies/topology_p5.svg)

## Objectif

Déployer la téléphonie IP sur le **VLAN 30 (voix)** avec **Cisco CallManager Express**, en couvrant le cycle de vie complet d'un poste : 

- Provisioning DHCP
- Config par TFTP (Option 150)
- Enregistrement SCCP
- Priorisation QoS

Chaque maillon **prouvé par une commande d'état ou un appel réel**, pas par un écran. 

Le fil directeur est une **chaîne** : DHCP → TFTP → SCCP → registration ; un maillon manquant casse tout, et le symptôme apparaît souvent trois étapes après la cause.
## Contrainte structurante 

- Le CME vit sur **DIST-SW1**, déjà HSRP Active + root STP du VLAN 30. Le service suit son Active. 
- Depuis que P4 a passé le Core en full-L3, **le VLAN 30 n'existe plus sur le Core** : la voix est ancrée sur la Distribution, ce qui rend ce placement non négociable. 
- Second pilier : **une seule autorité DHCP par domaine**. HQ-Router pour VLAN 10/20, **CME pour VLAN 30**. La preuve n'est pas que le CME distribue des baux, c'est qu'**aucun autre serveur** ne répond sur le VLAN 30.
## Décision de continuité (héritée de P3)

- La boucle ASA est déjà tuée en P3 (résumé `/20` + Null0). 
- P5 **vérifie**, ne reconfigure pas.
## Couverture CompTIA Network+

|Domaine|Concepts couverts|Statut|
|---|---|---|
|🌐 Services réseau|VoIP inter-switch · DHCP option 150 (TFTP) · autorité DHCP unique par domaine de broadcast · segmentation du pool (IPAM)|✅ 1001↔1002 `Connected` · HQ (10/20) / CME (30)|
|🔌 Commutation|Voice VLAN + annonce CDP · 802.1Q voix / data untagged sur un câble|✅ `Voice VLAN: 30` sur ACC|
|🎚️ QoS|Marquage DSCP EF (46) · frontière de confiance conditionnelle|✅ `trust device cisco-phone`|
|📜 Protocoles|SCCP (Skinny) TCP 2000 · TFTP · CDP|✅ `REGISTERED in SCCP`|
|🧭 Routage|Routes alignées sur l'espace alloué|✅ résumé `/20`|
|🛡️ Sécurité|`ip tftp source-interface` (asymétrie TFTP)|📋 documenté — réf. prod, non testable en PT|

## Conclusion

La VoIP m'a appris le raisonnement en chaîne de dépendances : DHCP → Option 150 → TFTP → SCCP est une chaîne où un seul maillon manquant casse tout le poste/.

Comprendre cet ordre, c'est ~90 % du dépannage VoIP. 

Second acquis, plus large que la voix : « une seule autorité DHCP » ne veut pas dire _un_ serveur pour tout le réseau (ce serait un SPOF), mais **une autorité par domaine de broadcast**, et la preuve n'est pas que le serveur distribue des baux, c'est qu'**aucun autre** ne répond sur le segment

---

[Partie 4 — Datacenter](../P4/README.md) ·  **Suivant : [Partie 6 — Wi-Fi](../P6/README.md)** — WLC + APs lightweight (CAPWAP), SSID WPA2 Corp/Guest, HSRPv2 VLAN 300, architecture hybride assumée (data plane par AP autonome). · [Vue d'ensemble du projet](../README.md)
