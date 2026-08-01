# Partie 5 — Téléphonie IP : CME, DHCP Option 150, QoS voix

**Bloc :** TheBigOffice · Services voix · **Outil :** Cisco Packet Tracer (CME 2811, DIST 3560-24PS, postes 7960) · **Certification :** CompTIA Network+
**Statut :** ✅ Validé — voix VLAN 30 opérationnelle, **CME co-localisé avec l'HSRP Active + root STP** (décision A1), **DHCP + Option 150 servis par une autorité unique** (le CME, prouvé par l'absence de pool VLAN 30 ailleurs), **appel inter-poste 1001↔1002 `Connected`**, QoS trust conditionnel lié au CDP, résumé ASA `/20` confirmé sûr · 3 incidents corrigés · 0 déviation · 6 dettes portées (D1–D6)

> **TheBigOffice — Partie 5 · Téléphonie IP**
> CME sur **DIST-SW1** (Active + root VLAN 30) · `Fa0/0 = 192.168.30.254` · DHCP `VOIP_PHONES` baux `.50-.99` + `option 150 → .254` · `telephony-service ip source-address .254 port 2000` · `ephone-dn 1001/1002` liés par `button 1:1 / 1:2` · postes 7960 sur ACC `Fa0/5` en data 10 + voice 30 · QoS `trust dscp` + `trust device cisco-phone`.
> Plan d'adressage complet → [`IPAM.md`](../IPAM.md).

---

## Objectif

Déployer la téléphonie IP sur le **VLAN 30 (voix)** avec **Cisco CallManager Express**, en couvrant le cycle de vie complet d'un poste : provisioning DHCP, config par TFTP (Option 150), enregistrement SCCP, priorisation QoS — chaque maillon **prouvé par une commande d'état ou un appel réel**, pas par un écran. Le fil directeur est une **chaîne** : DHCP → TFTP → SCCP → registration ; un maillon manquant casse tout, et le symptôme apparaît souvent trois étapes après la cause.

**Contrainte structurante.** Le CME vit sur **DIST-SW1** — déjà HSRP Active + root STP du VLAN 30. Le service suit son Active. Depuis que P4 a passé le Core en full-L3, **le VLAN 30 n'existe plus sur le Core** : la voix est ancrée sur la Distribution, ce qui rend ce placement non négociable. Second pilier : **une seule autorité DHCP par domaine** — HQ-Router pour VLAN 10/20, **CME pour VLAN 30**. La preuve n'est pas que le CME distribue des baux, c'est qu'**aucun autre serveur** ne répond sur le VLAN 30.

**Décision de continuité (héritée de P3).** La boucle ASA est déjà tuée en P3 (résumé `/20` + Null0). P5 **vérifie**, ne reconfigure pas.

![Topologie P5](../assets/topologies/topology_p5.svg)

---

## Niveaux & équipements

| Rôle | Équipement | Rôle dans la partie |
|---|---|---|
| Agent d'appel + DHCP + TFTP | **CME-Router** (2811) — *nouveau* | `telephony-service` (SCCP), pool DHCP VLAN 30, serveur TFTP (option 150). Un seul boîtier pour les trois. |
| Distribution (hôte du service) | **DIST-SW1** (3560-24PS) | Reçoit le CME sur `Fa0/10` (access VLAN 30). Déjà HSRP Active + root STP du VLAN 30. |
| Access (postes) | **ACC-SW1 / ACC-SW2** | Port `Fa0/5` en data VLAN 10 + voice VLAN 30 + QoS trust conditionnel. |
| Postes | **IP Phone 1001 / 1002** (7960) — *nouveaux* | S'enregistrent en SCCP, un PC derrière (un câble, deux VLAN). |
| Périmètre (contrôle) | **ASA / HQ-Router** | Non reconfigurés — on **vérifie** le résumé `/20` sûr et l'absence de pool VLAN 30 sur le HQ. |

Tout le campus P1/P2, le périmètre P3 et le datacenter P4 sont **inchangés** — P5 n'ajoute que le CME, deux postes et la config voix des ports. Câblage détaillé → [WORKFLOW P5](./WORKFLOW.md).

---

## Couverture CompTIA Network+

| Domaine | Concept | Statut |
|---|---|---|
| Services réseau | VoIP — appels inter-switch | ✅ 1001↔1002 `Connected` |
| Services réseau | DHCP Option 150 (TFTP) | ✅ baux `.50/.51` + opt.150 `.254` |
| Services réseau | Autorité DHCP unique par domaine de broadcast | ✅ HQ (10/20) / CME (30) |
| Services réseau | IPAM — segmentation du pool | ✅ exclusions `.1-.49` + `.100-.254` |
| Commutation | Voice VLAN + annonce CDP | ✅ `Voice VLAN: 30` sur ACC |
| Commutation | Tag 802.1Q voix / data untagged (un câble) | ✅ |
| QoS | DSCP EF (46) + latence/gigue/perte | ✅ marquage voix |
| QoS | Frontière de confiance conditionnelle | ✅ `trust device cisco-phone` |
| Protocoles | SCCP (Skinny) TCP 2000 · TFTP · CDP | ✅ `REGISTERED in SCCP` |
| Routage | Routes alignées sur l'espace alloué | ✅ résumé `/20` sûr |
| Sécurité | `ip tftp source-interface` (asymétrie TFTP) | 📋 Documenté (réf. prod, non testable PT) |

---

## Matrice de validation (locale)

> Chaque `✅` cite un `[P-##]` de l'**[Annexe — Captures de preuve du WORKFLOW P5](./WORKFLOW.md#annexe--captures-de-preuve)**.

| ✅ Prouvé (par résultat, appel ou état) | ⚠️ Configuré / limité par PT |
|---|---|
| **Appel inter-poste : 1001 → 1002 = `Connected`** (`Ring Out` → `From:1001 ringing` → `Connected`) — [P-01](./WORKFLOW.md#p-01), [P-02](./WORKFLOW.md#p-02), [P-03](./WORKFLOW.md#p-03) | LB / media stream avancé : hors périmètre (téléphonie basique) |
| Enregistrement SCCP : `show ephone` = **`REGISTERED in SCCP`**, IP `.50`/`.51`, `button 1: dn 1/2 … IDLE` — [P-04](./WORKFLOW.md#p-04) | `ip tftp source-interface Loopback0` : non supporté PT (dette D1/D2, réf. prod) |
| Liaison bouton→DN : `show run \| section ephone` = `button 1:1` / `1:2` (corrige I-2) — [P-05](./WORKFLOW.md#p-05) | |
| DHCP mono-autorité (VLAN 30) : `show ip dhcp binding` sur le CME = baux `.50`/`.51` uniquement — [P-06](./WORKFLOW.md#p-06), [P-06b](./WORKFLOW.md#p-06b) | |
| Preuve d'absence de 2e serveur : HQ-Router `section dhcp` = pools 10/20 seulement, aucun `192.168.30.x` — [P-07](./WORKFLOW.md#p-07) | |
| Placement du service : DIST-SW1 `show standby brief` = `Vl30 … 110 P Active` — [P-08](./WORKFLOW.md#p-08) | |
| Audit résumé : ASA `show route` = `S 10.0.0.0 255.255.240.0` (un seul `/20`, aucun `/8`/`/16`) — [P-09](./WORKFLOW.md#p-09) | |
| Voice VLAN + tag : ACC-SW1 `show interfaces Fa0/5 switchport` = `Access 10 / Voice 30` — [P-10](./WORKFLOW.md#p-10), [P-10c](./WORKFLOW.md#p-10c) | |
| QoS trust conditionnel : ACC-SW1 `show mls qos interface Fa0/5` = `trust device: cisco-phone` — [P-11](./WORKFLOW.md#p-11) | |
| Liaison CME : `Fa0/0 .254 up/up`, `ping .1` 5/5, `show arp` `.1` résolu — [P-12a](./WORKFLOW.md#p-12a), [P-12b](./WORKFLOW.md#p-12b), [P-12c](./WORKFLOW.md#p-12c) | |

> Un ou plusieurs `Request timed out` sur le **premier** paquet d'un flux frais (ARP/build) ne sont pas une faute. Un poste plus long sur `Configuring CM List` après `reset all` est un délai d'inscription (30-60 s), à distinguer du blocage permanent de I-2 (`button` manquant).

---

## Registre d'erreurs & dette technique

> Registre complet — décisions & pièges navigués (A1–B4), **incidents de session résolus (I-1 `Fa0/0` shutdown, I-2 `button` manquant, I-3 quarantaine 998)**, et dettes portées (D1–D6) — **source unique : [WORKFLOW P5 §5](./WORKFLOW.md)**. Non dupliqué ici.

---

⬅️ [Partie 4 — Datacenter](../P4/README.md) · ⬆️ **Suivant : [Partie 6 — Wi-Fi](../P6/README.md)** — WLC + APs lightweight (CAPWAP), SSID WPA2 Corp/Guest, HSRPv2 VLAN 300, architecture hybride assumée (data plane par AP autonome). · [Vue d'ensemble du projet](../README.md)
