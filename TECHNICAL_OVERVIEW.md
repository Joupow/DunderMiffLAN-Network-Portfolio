# TheBigOffice — Vue d'ensemble technique

---

## Sommaire

- [Objectif et intention](#objectif-et-intention)
- [Comment lire ce dépôt](#comment-lire-ce-dépôt)
- [Ce qu'un reviewer challengera](#ce-quun-reviewer-challengera)
- [Résumé technique par partie](#résumé-technique-par-partie)
- [Décisions de conception structurantes](#décisions-de-conception-structurantes)
- [Fils transversaux](#fils-transversaux)
- [Écarts de production](#écarts-de-production)
- [Preuve anti-copier-coller](#preuve-anti-copier-coller)

---

## Objectif et intention

Le lab simule un petit réseau d'entreprise en six blocs fonctionnels, chacun s'appuyant sur une base validée par le précédent :

1. **P1 — LAN siège** : fondations de commutation.
2. **P2 — Routage & redondance** : HSRP, OSPF, DHCP.
3. **P3 — Périmètre** : pare-feu ASA, DMZ, NAT.
4. **P4 — Datacenter** : fabric Spine-Leaf routée.
5. **P5 — Voix** : téléphonie CME.
6. **P6 — Wi-Fi** : contrôleur WLC et APs.

L'objectif est de démontrer comment **commutation, routage, sécurité, services, haute disponibilité et dépannage** interagissent dans un environnement cohérent unique. L'état décrit est l'état **as-built** : chaque partie livre une base propre validée avant la suivante, et les décisions structurantes sont justifiées, pas seulement appliquées.

---

## Comment lire ce dépôt

Trois niveaux de documentation, avec **un domicile canonique par type de contenu**. Cette règle est ce qui empêche deux documents de se contredire ; la respecter est plus important que la joliesse de chaque page.

| Type de contenu | Domicile canonique | Renvois (ne redéfinissent pas) |
|---|---|---|
| Décisions + *pourquoi* d'une partie | `README_PX` — Objectif, Contrainte structurante, Décision de continuité, Conclusion | cet overview (carte transversale) |
| Couverture CompTIA Network+ + statut ✅/⚠️/❌ | `README_PX` — Couverture CompTIA Network+ | statut **dérivé** de la validation WORKFLOW §3, jamais saisi en parallèle |
| Plan d'adressage complet | [`IPAM.md`](./IPAM.md) | README (lien d'en-tête + valeurs clés inline : VIP, physiques) |
| Cadrage (topologie as-built, niveaux & équipements) | `WORKFLOW_PX` §1 | README (topologie logique) |
| CLI pas-à-pas d'une partie | `WORKFLOW_PX` §2 — Étapes de configuration | README (objectif de chaque chantier) |
| Preuves de validation + dépannage | `WORKFLOW_PX` §3 — Validation de bout en bout, Dépannage | README (statut dans la table de couverture) |
| Registre corrections + dette + limites PT | `WORKFLOW_PX` §3 — Registre d'erreurs & dette technique | cet overview (agrégat prod uniquement) |
| Écarts de production consolidés | **cet overview** — [Écarts de production](#écarts-de-production) | README / WORKFLOW (lien) |

En résumé d'adressage : l'espace utilisateur/voix/Wi-Fi est en `192.168.0.0/16`, la DMZ et le datacenter en `172.16.0.0/16`, le transit interne est un bloc contigu `10.0.0.0/20` découpé en `/30` point-à-point, et le périmètre utilise des plages publiques de test (RFC 5737). Le détail — VLAN, liens de transit, Router-IDs, autorités DHCP par domaine — vit dans [`IPAM.md`](./IPAM.md).

---

## Ce qu'un reviewer challengera

**« Pourquoi Packet Tracer ? »**
PT sert à pratiquer et documenter des concepts de niveau Network+ dans une topologie simulée cohérente. Il démontre la logique de configuration mais ne modélise pas les fonctionnalités avancées. Le projet s'arrête à la Partie 6 parce qu'au-delà, chaque sujet demanderait GNS3 / Cisco CML ou de l'équipement physique pour valider un comportement réel. Cette limite est **documentée, pas cachée**.

**« Est-ce prêt pour la production ? »**
Non, et les écarts sont suivis explicitement ([Écarts de production](#écarts-de-production)). C'est un lab de portfolio junior ; l'honnêteté sur les limites est le levier de crédibilité.

**« Qu'est-ce qui prouve que ce n'est pas du copier-coller ? »**
Trois choses qu'un copier-coller ne produit pas : le raisonnement de conception est **justifié** ([Décisions de conception structurantes](#décisions-de-conception-structurantes)) ; les **incidents de build sont documentés** comme apprentissages ([Preuve anti-copier-coller](#preuve-anti-copier-coller)) ; les **matrices de validation sont honnêtes** — chaque partie distingue ✅ prouvé / ⚠️ configuré non prouvé / ❌ non simulable, et un flux bloqué se prouve par le **compteur d'ACL**, jamais par un timeout.

---

## Résumé technique par partie

Index du portfolio. Le détail vit dans le `README_PX` et le `WORKFLOW_PX` de chaque ligne.

| # | Bloc | Équipements principaux | Concepts principaux | Point notable |
|---|---|---|---|---|
| P1 | LAN siège | Core 3650, 2× Distribution 3560, 4× Access 2960 | VLANs, trunks, Rapid PVST+, VLAN de management, natif 999 trou noir | Roots STP sur la Distribution, alignés sur le plan HSRP de P2 ; aucun Access n'est root |
| P2 | Routage & redondance | Core, Distribution, HQ-Router | HSRP réparti, OSPF `/30`, DHCP par domaine + relais unique | Relais DHCP pointant le HQ-Router (`10.0.1.2`), pas le Core ; Active HSRP réparti par VLAN |
| P3 | Périmètre | ASA 5506-X, routeur FAI, services DMZ | Security-levels, 3 ACL opposées, NAT/PAT, IDS/SPAN, port-security | Route par défaut originée dans OSPF ; résumés internes verrouillés par Null0 AD 254 (côté HQ-Router) |
| P4 | Datacenter | Spines / Leafs / Border Leafs — tous 3650 | Spine-Leaf routée, fabric OSPF, tiers app + stockage, VIP LB | Deux Border Leafs sur le Core (ECMP) ; fabric entrée dans le `/20` sans renumérotation |
| P5 | Voix | CME, postes IP, switches d'accès | DHCP Option 150, SCCP, TFTP, frontière QoS | CME sur DIST-SW1 (Active + root du VLAN 30) ; héritage P3 vérifié — résumé `/20` audité sûr, aucune reconfig |
| P6 | Wi-Fi | WLC, APs lightweight, AP autonome | CAPWAP, SSID Corp 301 / Guest 310, WPA2, HSRPv2 VLAN 300 | VLAN 300 consolidé sur DIST-SW1 (Active + root + DHCP + WLC) ; contrôle et data plane prouvés séparément |

---

## Décisions de conception structurantes

Carte des choix qui traversent le lab. Chaque ligne est développée dans le `README_PX` correspondant.

| Domaine | Décision | Pourquoi elle compte |
|---|---|---|
| 🧭 Routage | Router-ID OSPF codés en dur | Évite un comportement d'adjacence instable |
| 🧭 Routage / 🔒 Sécurité | OSPF tenu hors du VLAN de management | Réduit l'exposition du plan de contrôle |
| 🧭 Routage / 🏷️ IPAM | Transit planifié en un `/20` contigu | La fabric datacenter entre dans le plan sans renumérotation de bout en bout |
| 🧭 Routage / 🔒 Sécurité | Résumés inside de l'ASA (`/16` + `/20`) verrouillés par des rejets Null0 (AD 254) sur le HQ-Router | Neutralise la boucle vers l'espace non alloué sans énumérer chaque `/30` ; le LPM fait toujours gagner une vraie route OSPF, Null0 n'attrape que les trous (voir [Fils transversaux](#fils-transversaux)) |
| 🔁 Haute disponibilité / 🧭 Routage | Passerelles HSRP réparties sur la Distribution | Supprime le SPOF de passerelle et étale la charge (VLANs lourds sur des switches différents) |
| 🔁 Haute disponibilité / 🌐 Services | Root STP + Active HSRP + service critique co-localisés par VLAN | Supprime le hairpin sur le lien inter-Distribution ; un service critique suit son Active |
| 🌐 Services | Une autorité DHCP par domaine de broadcast | Supprime les réponses DHCP dupliquées ou ambiguës |
| 🖥️ Datacenter | Deux Border Leafs sur le Core (fabric tout en 3650) | Sépare N-S de E-O et fournit l'ECMP ; le 3650 apporte 4 downlinks par Spine |
| 📶 Wi-Fi | Séparation plan de contrôle / plan de données au WLC | Évite de revendiquer une validation que PT ne peut pas prouver |

---

## Fils transversaux

### 1. L'espace transit `/20`, les résumés ASA et la boucle (P2 → P3 → P4 → P5)

Le transit interne est un bloc contigu `10.0.0.0/20` découpé en liens `/30` point-à-point. 

Côté ASA, l'inside est décrit par **deux résumés** : `192.168.0.0/16` (utilisateurs / voix / Wi-Fi) et `10.0.0.0/20` (transit), plutôt que par une multitude de `/30`.

Un résumé large couvre de l'espace **non alloué**. Un paquet vers une adresse inexistante (`192.168.150.1`, `10.0.14.x`) rebondissait ASA↔HQ-Router jusqu'à expiration du TTL — symptôme silencieux : pas d'erreur, juste du routage faux par intermittence.

**Résolution (P3, vérifiée en P5)** : chaque résumé est adossé à un **rejet Null0 flottant (AD 254) sur le HQ-Router** (`ip route 192.168.0.0/16 Null0 254`, `ip route 10.0.0.0/20 Null0 254`). 
Le *longest-prefix match* fait toujours gagner une vraie route OSPF ; le Null0 n'attrape que les trous. P5 **ne reconfigure pas**, il audite (`show route` → un seul `S 10.0.0.0/20`, aucun `/8`/`/16`) et confirme que le résumé ne résout que vers de l'espace alloué.

La fabric datacenter (P4) est le corollaire : parce que le transit était planifié en `/20` contigu, elle s'adresse dans `10.0.5+` et **entre dans le résumé sans renumérotation** — aucun `/30` existant ne bouge. Récit canonique et CLI : `WORKFLOW_P3` (étape 3) et `WORKFLOW_P5` (étape 0).

> **Leçon** : un résumé est dangereux dès qu'il inclut de l'espace non alloué. Plutôt qu'énumérer chaque `/30`, la conception garde les résumés compacts et pose un filet Null0 par bloc résumé. L'exposition résiduelle (dette D5) se solde en production par **OSPF sur l'ASA**, pour qu'il n'apprenne que des préfixes réellement annoncés.

### 2. La co-localisation Root STP + Active HSRP + service (P1 → P2 → P5 → P6)

Un même principe traverse quatre parties : **placer le root STP, l'Active HSRP et le service critique sur le même switch, par VLAN**, pour qu'un flux ne fasse jamais un détour par le lien inter-Distribution. Le split est fixé une fois pour toutes : **DIST-SW1 = Active + root `{10,30}`**, **DIST-SW2 = `{20,99}`**.

- **P1** fixe les roots STP sur la Distribution selon ce split, en **anticipant** le plan HSRP de P2 ; aucun Access ne peut gagner l'élection.
- **P2** aligne l'Active HSRP dessus sans re-toucher STP, et ordonne la bascule *Core d'abord* pour que la VIP `.1` ne soit jamais revendiquée par deux boîtiers.
- **P5** pose le CME sur **DIST-SW1** (déjà Active + root du VLAN 30). Depuis que P4 a passé le Core en full-L3, **le VLAN 30 n'existe plus sur le Core** : la voix est ancrée sur la Distribution, ce qui rend ce placement non négociable.
- **P6** consolide le VLAN 300 sur DIST-SW1 (Active + root + DHCP + WLC) et **ne touche jamais au VLAN 30** — rejouer un side-fix qui basculerait son root vers DIST-SW2 casserait la co-localisation CME.
- 

C'est une décision unique, honorée sur quatre parties — pas quatre décisions séparées.

---

## Écarts de production

> **Source unique du dépôt.** Les `README_PX` et `WORKFLOW_PX` renvoient ici ; ils ne redéfinissent pas leurs propres tables d'écarts prod. Chaque écart est un choix de périmètre lab assumé, avec sa direction de durcissement.

| Écart | État actuel (lab, as-built) | Direction production |
|---|---|---|
| Authentification OSPF | Non implémentée | Authentification des adjacences sur les liens routés |
| Plan de management | SSH/AAA partiels ; pas d'IP de management dédiée (managé in-band, Loopback0 différé) | SSHv2, AAA, ACL de management, journalisation centralisée, Loopback0 |
| Redondance DHCP | Une autorité par domaine ; relais mono-chemin (helper côté Active seul), pool sur l'Active — pas de nouveau bail si l'Active tombe | Split-scope / HA, miroir sur le Standby |
| Routage ASA | Résumés statiques inside (`/16` + `/20`) + verrous Null0 (AD 254) sur le HQ-Router ; audités sûrs (P5) | Routage dynamique (OSPF) sur le pare-feu |
| Data plane Wi-Fi | Chemin client CAPWAP non simulé (prouvé par AP autonome) | Vrai WLC, Cisco CML, GNS3 ou lab physique |
| Isolation Guest (VLAN 310) | Configurée, non prouvée en data plane (pas de portail captif PT) | Guest anchor WLC + portail captif |
| Load balancer datacenter | VIP présentée, round-robin conceptuel (pas de moteur LB en PT) | LB fonctionnel (HAProxy / F5), health-checks |
| Contrôle VoIP | Point CME/TFTP unique | Contrôle d'appel + services TFTP redondants (Loopback) |
| Passerelle datacenter | Un SVI par Leaf (SPOF par tier) | Anycast Gateway via VXLAN EVPN |

---

## Preuve anti-copier-coller

Incidents de build documentés comme apprentissages. Registres détaillés dans les `WORKFLOW_PX` (§3) ; cette table est l'agrégat transversal.

| Incident (partie)                                    | Impact                                            | Diagnostic / conception retenue                                                                                                    |
| ---------------------------------------------------- | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Downlinks Distribution laissés en access VLAN 1 (P1) | `%CDP-4-NATIVE_VLAN_MISMATCH`, VLANs taggés jetés | Repéré par `show interfaces trunk` ; trunk natif 999 appliqué aux `Fa0/1-4`                                                        |
| BPDU Guard posé sur un uplink inter-switch (P1)      | Port err-disable à la 1ʳᵉ BPDU légitime           | BPDU Guard réservé aux ports hôtes ; la protection de boucle sur uplinks est le travail de STP (`Altn BLK`)                        |
| Résumés ASA vers de l'espace non alloué (P3)         | Boucle ASA ↔ HQ jusqu'au TTL                      | Un verrou Null0 (AD 254) par bloc résumé sur le HQ-Router ; le LPM fait gagner une vraie route OSPF, Null0 n'attrape que les trous |
| Réponses DHCP dupliquées possibles (P5)              | Conflits de baux                                  | Une autorité DHCP par domaine de broadcast, prouvée par l'**absence** de tout second serveur sur le segment                        |
| Data plane client CAPWAP non simulable en PT (P6)    | Forwarding client invalidable                     | Plans de contrôle et de données prouvés **séparément** : WLC (4 APs `Online`) + AP autonome (client réel, TTL 127)                 |

---

⬆️ [Sommaire](#sommaire)
