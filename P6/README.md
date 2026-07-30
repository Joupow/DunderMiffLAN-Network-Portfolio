# Partie 6 — Réseau sans fil d'entreprise : WLC, CAPWAP, SSID/VLAN, HSRPv2 VLAN 300

**Bloc :** TheBigOffice · Services Wi-Fi · **Outil :** Cisco Packet Tracer 9.0 (Generic WLC, 4× LAP, 1× AP autonome, WLC 3504 = réf.) · **Certification :** CompTIA Network+
**Statut :** ✅ Validé — **4 APs `Online` en CAPWAP** (contrôle prouvé), **data plane client prouvé de bout en bout par l'AP autonome** (Wi-Fi → filaire VLAN 10, TTL 127), **HSRPv2 VLAN 300 opérationnel** (DIST-SW1 Active + root, DIST-SW2 Standby stabilisé), **DHCP VLAN 300 mono-autorité** (DIST-SW1, WLC DHCP off), **VLAN 30 voix (P5) non régressé** (décision A1 tenue) · 3 incidents corrigés · 5 déviations actées · 14 limitations PT documentées

> **TheBigOffice — Partie 6 · Wi-Fi**
> Generic WLC `192.168.100.200` sur **DIST-SW1 `Fa0/5`** (access VLAN 300) · VIP HSRPv2 `192.168.100.1` · pool DHCP `VLAN300` baux `.10-.50` · 4× LAP sur **ACC-SW1→4 `Fa0/7`** (access 300 + PortFast + BPDU Guard) · AP autonome sur **ACC-SW1 `Fa0/6`** (`TheBigOffice-Corp-Auto`, 2.4 GHz ch.6, WPA2-PSK) · WLANs `Corp` (301) + `-Guest` (310), WPA2-PSK/AES, **Local switching**.
> Plan d'adressage complet → [`IPAM.md`](../IPAM.md).

---

## Objectif

Poser une couche Wi-Fi d'entreprise sur la fondation filaire P1–P5 : architecture à contrôleur (WLC + APs lightweight), segmentation SSID/VLAN, WPA2, et passerelle redondée par **HSRPv2 sur le VLAN 300** — chaque maillon **prouvé par un état ou un trafic réel**. C'est cette partie qui introduit le VLAN 300 sur le réseau.

**Contrainte structurante.** Packet Tracer 9.0 **ne simule pas le data plane CAPWAP centralisé**. D'où une **architecture hybride assumée** : le contrôle est prouvé par le WLC (4 APs `Online`), le data plane client par un **AP autonome** (pont direct, sans contrôleur dans le chemin). Ce n'est pas un contournement honteux — c'est le seul moyen honnête de démontrer les deux plans dans l'outil.

Trois SSID / VLAN structurent la couche radio :

| SSID | VLAN | Sécurité | Rôle |
|---|---|---|---|
| `TheBigOffice-Corp` | 301 | WPA2-PSK / AES | Corp (via LAP, CAPWAP) |
| `TheBigOffice-Guest` | 310 | WPA2-PSK / AES | Guest (via LAP, CAPWAP) |
| `TheBigOffice-Corp-Auto` | 300 (pont) | WPA2-PSK / AES | AP autonome — **seul data plane client fonctionnel en PT** |

**Décision de continuité (héritée de P5).** DIST-SW1 est déjà HSRP Active + root STP des VLANs 10/30 ; le VLAN 300 suit. **On ne touche jamais au VLAN 30** : rejouer un side-fix qui basculerait son root casserait la co-localisation CME/Active de la décision A1. Vérifié après coup ([P-12]).

![Topologie P6](../assets/topologies/topology_p6.svg)

---

## Niveaux & équipements

| Rôle | Équipement | Rôle dans la partie |
|---|---|---|
| Contrôleur Wi-Fi | **Generic WLC** — *nouveau* | Enregistre 4 LAP en CAPWAP, diffuse Corp/Guest. Mgmt `.200`. Config 100 % GUI. |
| Contrôleur (réf.) | **WLC 3504** — *nouveau, déconnecté* | Réf. prod `.201`. Pas de config WLAN en PT → **jamais câblé**. |
| Distribution (hôte) | **DIST-SW1** (3560) | WLC sur `Fa0/5` (300). SVI 300 Active + root. Pool DHCP `VLAN300`. **VLAN 30 inchangé.** |
| Distribution (redondance) | **DIST-SW2** (3560) | SVI 300 Standby. **Inchangé sinon.** |
| Access (APs) | **ACC-SW1 → ACC-SW4** | `Fa0/7` = LAP (300 + hardening). `Fa0/6` (SW1) = AP autonome. `Fa0/5` = **poste voix P5, intouché.** |
| APs lightweight | **4× LAP** (3702i-class) — *nouveaux* | CAPWAP au WLC `.200`. Baux `.10-.13`. |
| AP autonome | **Access Point0** (AP-PT) — *nouveau* | `TheBigOffice-Corp-Auto`, 2.4 GHz ch.6, WPA2-PSK. Seul data plane client. |
| Client de test | **Laptop0** (WPC300N) | DHCP `.14`, prouve Wi-Fi → filaire. |

Tout le campus P1–P5 est **inchangé** — P6 n'ajoute que les VLANs 300/301/310, le WLC, les APs et le client. Câblage détaillé → [WORKFLOW P6](./WORKFLOW.md).

---

## Couverture CompTIA Network+

| Domaine | Concept | Statut |
|---|---|---|
| Sans fil | Access Point / WLC | ✅ 4 LAP + AP autonome + WLC |
| Sans fil | Autonome vs Lightweight | ✅ Différence data plane prouvée |
| Sans fil | CAPWAP | ✅ Tunnel établi, 4 APs `Online` |
| Sans fil | SSID / BSSID / ESSID | ✅ Corp + Guest diffusés |
| Sans fil | WPA2-PSK (AES) | ✅ Les deux SSID + autonome |
| Sans fil | Canal 2.4 GHz non-recouvrant | ✅ **Canal 6** (⚠️ downgrade vs 5 GHz ch.36 — dette L14) |
| Sans fil | WPA3 / 6 GHz / band steering | ⚠️ Non supportés PT |
| Sans fil | PSK vs Enterprise (802.1X) | ⚠️ PSK fait, Enterprise documenté |
| Sans fil | Réseau Guest | ⚠️ Configuré, isolation non testée en data plane |
| Sans fil | AP Groups | ✅ default-group, 2 WLANs, 4 APs |
| Commutation | Segmentation VLAN Wi-Fi | ⚠️ 300 validé ; 301/310 définis (pas sur le fil en PT) |
| Commutation | 802.1Q trunking · STP / PortFast / BPDU Guard | ✅ root VLAN 300 + ports AP durcis |
| Routage | Inter-VLAN sans fil | ✅ 300 → 10 via AP autonome, TTL 127 |
| Haute disponibilité | HSRPv2 passerelle Wi-Fi | ✅ Groupe 300, Active/Standby prouvés |
| Services IP | DHCP pour APs | ✅ Baux `.10-.13` |

---

## Matrice de validation (locale)

> Chaque `✅` cite un `[P-##]` de l'**[Annexe — Captures de preuve du WORKFLOW P6](./WORKFLOW.md)**.

| ✅ Prouvé (par état, appel ou trafic) | ⚠️ Configuré / limité par PT |
|---|---|
| **Enregistrement CAPWAP : 4 LAP `Online`**, MACs `.10-.13` = baux DHCP — [P-18], [P-06] | Data plane client **via WLC** (301/310) : droppé en PT (L5) |
| **Data plane client : laptop → `.100.1` (4/4) puis → `.10.52` (TTL 127)** via AP autonome — [P-20], [P-21] | DHCP Wi-Fi centralisé auto : APIPA en PT (L6) — contourné par AP autonome |
| Diffusion SSID Corp + Guest — [P-18], [P-18b] | Isolation Guest : non testée en data plane (captive portal ❌) |
| HSRPv2 VLAN 300 : Active `.100.1` + Standby stabilisé — [P-12], [P-13] | WPA3 / 6 GHz / band steering : non simulables |
| Root STP VLAN 300 exécuté : `This bridge is the root` — [P-15] | Segmentation 301/310 : définie, jamais sur le fil (port WLC access) |
| **VLAN 30 (voix P5) non régressé** : DIST-SW1 Active 30 — [P-12] | |
| DHCP mono-autorité VLAN 300 : baux `.10-.14`, WLC DHCP off — [P-06] | |
| Trunks liste complète, 301/310 confinés inter-DIST — [P-07], [P-08], [P-09] | |
| Voix P5 intacte : `Fa0/5` en 10/30, jamais écrasé par un AP — [P-01] | |
| WLC joignable : `ping .200` 5/5 — [P-16], [P-17] | |

> Un `Request timed out` sur le **premier** paquet d'un flux frais (ARP/build) n'est pas une faute. Le succès du ping laptop **prouve par élimination** le chemin AP autonome : via un LAP, PT dropperait le data plane.

---

## Registre d'erreurs & dette technique

> Registre complet — **incidents de session (I-1 collision `Fa0/5`, I-2 canal recouvrant, I-3 HSRP `Listen`)**, déviations actées (DV1–DV5), dettes portées (L14, D-HA, D-GUEST) et catalogue exhaustif des limitations PT (L1–L14) — **source unique : [WORKFLOW P6 §5](./WORKFLOW.md)**. Non dupliqué ici.

---

⬅️ [Partie 5 — Téléphonie IP](../P5/README.md) · ⬆️ [Vue d'ensemble du projet](../README.md) · *(fin de la série Packet Tracer — suite sur GNS3 : IPv6 avancé, monitoring, PKI/RADIUS)*
