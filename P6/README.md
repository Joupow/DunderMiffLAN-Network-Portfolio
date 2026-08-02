# Partie 6 : Wi-Fi d'entreprise 

 **Concepts clés** : WLC, CAPWAP, SSID/VLAN, HSRPv2 VLAN 300  · **Certification :** CompTIA Network+  · **Outil :** Cisco Packet Tracer 9.0

- Plan d'adressage complet → [`IPAM.md`](../IPAM.md)
- Progression étape par étape → [`WORKFLOW P6`](./WORKFLOW.md)

---

## Objectif

Poser une couche Wi-Fi d'entreprise sur la fondation filaire P1–P5 : 

- architecture à contrôleur (WLC + APs lightweight)
- segmentation SSID/VLAN, 
- WPA2, 
- et passerelle redondée par **HSRPv2 sur le VLAN 300** en **prouvant chaque maillon par un état ou un trafic réel**. 

C'est cette partie qui introduit le VLAN 300 sur le réseau.

**Contrainte structurante :** 

- Packet Tracer 9.0 **ne simule pas le data plane CAPWAP centralisé**. 
- Le WLC enregistre bien les APs (control plane prouvé), mais ne sait pas ré-injecter le trafic client par son port access. 
- D'où une **architecture hybride assumée** : le contrôle est prouvé par le WLC (4 APs `Online`), le data plane client par un **AP autonome** (pont direct, sans contrôleur dans le chemin). 

C'est le seul moyen honnête de démontrer les deux plans dans l'outil.

**Décision de continuité (héritée de P5) :** 

- DIST-SW1 est déjà HSRP Active + root STP des VLANs 10/30
- Le VLAN 300 suit cette logique. 
- **On ne touche jamais au VLAN 30** : rejouer un side-fix qui basculerait son root vers DIST-SW2 casserait la co-localisation CME/Active de la décision A1. Vérifié après coup.


---
## Topologie logique

![Topologie P6](../assets/topologies/topology_p6.svg)


---

## Couverture CompTIA Network+

| Domaine                | Concept                                       | Statut                                                  |
| ---------------------- | --------------------------------------------- | ------------------------------------------------------- |
| 📶 Sans fil            | Access Point / WLC                            | ✅ 4 LAP + AP autonome + WLC                             |
| 📶 Sans fil            | Autonome vs Lightweight                       | ✅ Différence data plane prouvée                         |
| 📶 Sans fil            | CAPWAP                                        | ✅ Tunnel établi, 4 APs `Online`                         |
| 📶 Sans fil            | SSID / BSSID / ESSID                          | ✅ Corp + Guest diffusés                                 |
| 📶 Sans fil            | WPA2-PSK (AES)                                | ✅ Les deux SSID + autonome                              |
| 📶 Sans fil            | Canal 2.4 GHz non-recouvrant                  | ✅ **Canal 6** (⚠️ downgrade vs 5 GHz ch.36 — dette L14) |
| 📶 Sans fil            | WPA3 / 6 GHz / band steering                  | ⚠️ Non supportés PT                                     |
| 📶 Sans fil            | PSK vs Enterprise (802.1X)                    | ⚠️ PSK fait, Enterprise documenté                       |
| 📶 Sans fil            | Réseau Guest                                  | ⚠️ Configuré, isolation non testée en data plane        |
| 📶 Sans fil            | AP Groups                                     | ✅ default-group, 2 WLANs, 4 APs                         |
| 🔌 Commutation         | Segmentation VLAN Wi-Fi                       | ⚠️ 300 validé ; 301/310 définis (pas sur le fil en PT)  |
| 🔌 Commutation         | 802.1Q trunking · STP / PortFast / BPDU Guard | ✅ root VLAN 300 + ports AP durcis                       |
| 🧭 Routage             | Inter-VLAN sans fil                           | ✅ 300 → 10 via AP autonome, TTL 127                     |
| 🔁 Haute disponibilité | HSRPv2 passerelle Wi-Fi                       | ✅ Groupe 300, Active/Standby prouvés                    |
| 🌐 Services            | DHCP pour APs                                 | ✅ Baux `.10-.13`                                        |

---
## Test & validation ✅

Cette section recense ce qui a été testé et validé : 

- Chaque `✅` cite un `[P-##]` de l'**[Annexe — Captures de preuve du WORKFLOW P6](./WORKFLOW.md#annexe--captures-de-preuve)**.
- Un enregistrement est prouvé par un **état** (`Online`, `root`), un flux par du **trafic réel** (`ping`, TTL), jamais par un timeout

| **📶 Sans fil**                 | Résultat / état                                   | Preuve                                                                                             |
| ------------------------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Enregistrement CAPWAP           | **4 LAP `Online`**, MACs `.10-.13` = baux DHCP    | [P-18](https://claude.ai/chat/WORKFLOW.md#p-18), [P-04](https://claude.ai/chat/WORKFLOW.md#p-04)   |
| Data plane client (AP autonome) | laptop → `.100.1` (4/4) puis → `.10.52` (TTL 127) | [P-20](https://claude.ai/chat/WORKFLOW.md#p-20), [P-21](https://claude.ai/chat/WORKFLOW.md#p-21)   |
| Diffusion SSID                  | Corp + Guest                                      | [P-18](https://claude.ai/chat/WORKFLOW.md#p-18), [P-18b](https://claude.ai/chat/WORKFLOW.md#p-18b) |
| WLC joignable                   | `ping .200` 5/5                                   | [P-16](https://claude.ai/chat/WORKFLOW.md#p-16), [P-17](https://claude.ai/chat/WORKFLOW.md#p-17)   |

| **🔁 Haute disponibilité (VLAN 300)** | Résultat / état                      | Preuve                                                                                           |
| ------------------------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------ |
| HSRPv2                                | Active `.100.1` + Standby stabilisé  | [P-12](https://claude.ai/chat/WORKFLOW.md#p-12), [P-13](https://claude.ai/chat/WORKFLOW.md#p-13) |
| Root STP                              | `This bridge is the root` (VLAN 300) | [P-15](https://claude.ai/chat/WORKFLOW.md#p-15)                                                  |

| 🔌 Commutation / VLAN** | Résultat / état                             | Preuve                                                                                                                                            |
| ----------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Trunks                  | liste complète, 301/310 confinés inter-DIST | [P-07](https://claude.ai/chat/WORKFLOW.md#p-07), [P-08](https://claude.ai/chat/WORKFLOW.md#p-08), [P-09](https://claude.ai/chat/WORKFLOW.md#p-09) |

| **🌐 Services (DHCP)**        | Résultat / état              | Preuve                                          |
| ----------------------------- | ---------------------------- | ----------------------------------------------- |
| DHCP mono-autorité (VLAN 300) | baux `.10-.14`, WLC DHCP off | [P-04](https://claude.ai/chat/WORKFLOW.md#p-04) |

| **📞 Voix — non-régression (P5)** |                                           |                                                 |
| --------------------------------- | ----------------------------------------- | ----------------------------------------------- |
| VLAN 30 non régressé              | DIST-SW1 Active 30                        | [P-12](https://claude.ai/chat/WORKFLOW.md#p-12) |
| Voix P5 intacte                   | `Fa0/5` en 10/30, jamais écrasé par un AP | [P-01](https://claude.ai/chat/WORKFLOW.md#p-01) |

### ⚠️ Configuré / limité par PT

|Point|Limite|Réf.|
|---|---|---|
|Data plane client **via WLC** (301/310)|droppé en PT|dette L5|
|DHCP Wi-Fi centralisé auto|APIPA en PT — contourné par AP autonome|dette L6|
|Isolation Guest|non testée en data plane (captive portal absent)|—|
|WPA3 / 6 GHz / band steering|non simulables en PT|—|
|Segmentation 301/310|définie, jamais sur le fil (port WLC en access)|—|

> Un `Request timed out` sur le **premier** paquet d'un flux frais (ARP/build) n'est pas une faute. Le succès du ping laptop **prouve par élimination** le chemin AP autonome : via un LAP, PT dropperait le data plane.

---

## Conclusion

Packet Tracer ne simule pas le data plane CAPWAP centralisé; et l'apprentissage n'est pas de le contourner en douce, mais de **nommer la limite honnêtement** : 

Contrôle prouvé par le WLC (4 APs `Online`), data plane prouvé par un AP autonome. Reconnaître ce qu'un outil _ne peut pas_ démontrer, et le documenter comme tel, est une preuve de maturité, pas un aveu de faiblesse. 

C'est aussi ce qui justifie la clôture du projet ici et le passage à un émulateur pour la suite. 

---

⬅️ [Partie 5 — Téléphonie IP](../P5/README.md) · ⬆️ [Vue d'ensemble du projet](../README.md) · *(fin de la série Packet Tracer — suite sur GNS3 : IPv6 avancé, monitoring, PKI/RADIUS)*
