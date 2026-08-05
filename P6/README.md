# Partie 6 : Wi-Fi  

 **Concepts clés** : WLC · APs lightweight · CAPWAP · SSID WPA2 Corp/Guest · HSRPv2 VLAN 300

- 💻**Outil** : Cisco Packet Tracer 9.0
- 🏷️ Plan d'adressage complet → [IPAM](../IPAM.md)
- 📝 Progression étape par étape → [WORKFLOW P6](./WORKFLOW.md)
- 🎓 **Certification :** CompTIA Network+
## Topologie logique

![Topologie P6](../assets/topologies/topology_p6.svg)

## Objectif

Poser une couche Wi-Fi d'entreprise sur la fondation filaire P1–P5 : 

- architecture à contrôleur (WLC + APs lightweight)
- segmentation SSID/VLAN, 
- WPA2, 
- et passerelle redondée par **HSRPv2 sur le VLAN 300** en **prouvant chaque maillon par un état ou un trafic réel**. 

C'est cette partie qui introduit le VLAN 300 sur le réseau.

## Contrainte structurante

- Packet Tracer 9.0 **ne simule pas le data plane CAPWAP centralisé**. 
- Le WLC enregistre bien les APs (control plane prouvé), mais ne sait pas ré-injecter le trafic client par son port access. 
- D'où une **architecture hybride assumée** : le contrôle est prouvé par le WLC (4 APs `Online`), le data plane client par un **AP autonome** (pont direct, sans contrôleur dans le chemin). 

C'est le seul moyen honnête de démontrer les deux plans dans l'outil.


## Décision de continuité (héritée de P5)

- DIST-SW1 est déjà HSRP Active + root STP des VLANs 10/30
- Le VLAN 300 suit cette logique. 
- **On ne touche jamais au VLAN 30** : rejouer un side-fix qui basculerait son root vers DIST-SW2 casserait la co-localisation CME/Active de la décision A1. Vérifié après coup.


## Couverture CompTIA Network+

|Domaine|Concepts couverts|Statut|
|---|---|---|
|📶 Sans fil : infra|AP/WLC · autonome vs lightweight · CAPWAP · AP groups|✅ 4 LAP + AP autonome + WLC · 4 APs `Online`|
|📶 Sans fil : diffusion & sécu|SSID/BSSID/ESSID · WPA2-PSK (AES) · canal 2.4 GHz non-recouvrant|✅ Corp + Guest · canal 6|
|🔌 Commutation|802.1Q trunking · STP/PortFast/BPDU Guard · segmentation VLAN Wi-Fi|✅ root VLAN 300 · ports AP durcis|
|🧭 Routage|Inter-VLAN sans fil|✅ 300 → 10 via AP autonome|
|🔁 Haute disponibilité|HSRPv2 passerelle Wi-Fi|✅ groupe 300, Active/Standby|
|🌐 Services|DHCP pour APs|✅ baux `.10-.13`|

**Limites assumées & dette (contrainte outil, pas défaut de conception)**

- 📋 WPA3 / 6 GHz / band steering · PSK-Enterprise (802.1X) : non supportés / non simulables en Packet Tracer, documentés comme référence prod.
- ⚠️ Réseau Guest : configuré, isolation non vérifiable en data plane sous PT.
- ⚠️ VLANs 301/310 : définis, pas exerçables sur le fil en PT (300 validé).
- 🔧 Canal 6 (2.4 GHz) : downgrade assumé vs 5 GHz ch.36 : **dette L14**.

## Conclusion

Packet Tracer ne simule pas le data plane CAPWAP centralisé; et l'apprentissage n'est pas de le contourner en douce, mais de **nommer la limite honnêtement** : 

Contrôle prouvé par le WLC (4 APs `Online`), data plane prouvé par un AP autonome. Reconnaître ce qu'un outil _ne peut pas_ démontrer, et le documenter comme tel, est une preuve de maturité, pas un aveu de faiblesse. 

C'est aussi ce qui justifie la clôture du projet ici et le passage à un émulateur pour la suite. 

---

⬅️ [Partie 5 : VoIP](../P5/README.md) · ⬆️ [Vue d'ensemble du projet](../README.md) · 🔁 [Workflow P6](./WORKFLOW.md) ·
