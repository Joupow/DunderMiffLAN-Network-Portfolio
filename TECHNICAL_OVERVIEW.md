# TheBigOffice — Vue d'ensemble technique

## Sommaire

- [Objectif et intention](#objectif-et-intention)
- [Comment lire ce dépôt](#comment-lire-ce-dépôt)
- [Ce qu'un reviewer challengera](#ce-quun-reviewer-challengera)
- [Résumé technique par partie](#résumé-technique-par-partie)
- [Décisions de conception structurantes](#décisions-de-conception-structurantes)
- [Fils transversaux](#fils-transversaux)
- [Écarts de production](#écarts-de-production)
- [Preuve anti-copier-coller](#preuve-anti-copier-coller)

## Objectif et intention

Le lab simule un petit réseau d'entreprise en six blocs fonctionnels, chacun s'appuyant sur une base validée par le précédent :

- **P1 : LAN siège** : fondations de commutation.
- **P2 : Routage & redondance** : HSRP, OSPF, DHCP.
- **P3 : Périmètre** : pare-feu ASA, DMZ, NAT.
- **P4 : Datacenter** : fabric Spine-Leaf routée.
- **P5 : Voix** : téléphonie CME.
- **P6 : Wi-Fi** : contrôleur WLC et APs.

L'objectif est de démontrer comment **commutation, routage, sécurité, services, haute disponibilité et dépannage** interagissent dans un environnement cohérent unique. 

L'état décrit est l'état **as-built** : chaque partie livre une base propre validée avant la suivante, et les décisions structurantes sont justifiées, pas seulement appliquées.

## Comment lire ce dépôt

Trois niveaux de documentation, avec **un domicile canonique par type de contenu**. 

| Type de contenu                                     | Domicile canonique                                                                  |
| --------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Décisions + *pourquoi* d'une partie                 | `README_PX` — Objectif, Contrainte structurante, Décision de continuité, Conclusion |
| Couverture CompTIA Network+ + statut ✅/⚠️/❌         | `README_PX` — Couverture CompTIA Network+                                           |
| Plan d'adressage complet                            | [`IPAM.md`](./IPAM.md)                                                              |
| Cadrage (topologie as-built, niveaux & équipements) | `WORKFLOW_PX` §1                                                                    |
| CLI pas-à-pas d'une partie                          | `WORKFLOW_PX` §2 — Étapes de configuration                                          |
| Preuves de validation + dépannage                   | `WORKFLOW_PX` §3 — Validation de bout en bout, Dépannage                            |
| Registre corrections + dette + limites PT           | `WORKFLOW_PX` §3 — Registre d'erreurs & dette technique                             |
| Écarts de production consolidés                     | **cet overview** — [Écarts de production](#écarts-de-production)                    |


## Ce qu'un reviewer challengera

**« Pourquoi Packet Tracer ? »**

PT sert à pratiquer et documenter des concepts de niveau Network+ dans une topologie simulée cohérente. Il démontre la logique de configuration mais ne modélise pas les fonctionnalités avancées. 

Le projet s'arrête à la Partie 6 parce qu'au-delà, chaque sujet demanderait GNS3 / Cisco CML ou de l'équipement physique pour valider un comportement réel. Cette limite est **documentée, pas cachée**.

**« Est-ce prêt pour la production ? »**

Non, et les écarts sont suivis explicitement ([Écarts de production](#écarts-de-production)). C'est un lab de portfolio junior ; l'honnêteté sur les limites est le levier de crédibilité.

**« Qu'est-ce qui prouve que ce n'est pas du copier-coller ? »**

Trois choses qu'un copier-coller ne produit pas : 

- Le raisonnement de conception est **justifié** ([Décisions de conception structurantes](#décisions-de-conception-structurantes)) ; 
- Les **incidents de build sont documentés** comme apprentissages ([Preuve anti-copier-coller](#preuve-anti-copier-coller)) ; 
- Les **matrices de validation sont honnêtes**. Chaque partie distingue ✅ prouvé / ⚠️ configuré non prouvé / ❌ non simulable, et un flux bloqué se prouve par le **compteur d'ACL**, jamais par un timeout.

## Résumé technique par partie

Index du portfolio. Le détail vit dans le `README_PX` et le `WORKFLOW_PX` de chaque ligne.

| #                    | Bloc                 | Équipements principaux                          | Concepts principaux                                                 | Point notable                                                                                                |
| -------------------- | -------------------- | ----------------------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [P1](./P1/README.md) | LAN 3 niveaux        | Core 3650, 2× Distribution 3560, 4× Access 2960 | VLANs, trunks, Rapid PVST+, VLAN de management, natif 999 trou noir | Roots STP sur la Distribution, alignés sur le plan HSRP de P2 ; aucun Access n'est root                      |
| [P2](./P2/README.md) | Routage & redondance | Core, Distribution, HQ-Router                   | HSRP réparti, OSPF `/30`, DHCP par domaine + relais unique          | Relais DHCP pointant le HQ-Router (`10.0.1.2`), pas le Core ; Active HSRP réparti par VLAN                   |
| [P3](./P3/README.md) | DMZ & Pare-feu       | ASA 5506-X, routeur FAI, services DMZ           | Security-levels, 3 ACL opposées, NAT/PAT, IDS/SPAN, port-security   | Route par défaut originée dans OSPF ; résumés internes verrouillés par Null0 AD 254 (côté HQ-Router)         |
| [P4](./P4/README.md) | Datacenter           | Spines / Leafs / Border Leafs — tous 3650       | Spine-Leaf routée, fabric OSPF, tiers app + stockage, VIP LB        | Deux Border Leafs sur le Core (ECMP) ; fabric entrée dans le `/20` sans renumérotation                       |
| [P5](./P5/README.md) | VoIP                 | CME, postes IP, switches d'accès                | DHCP Option 150, SCCP, TFTP, frontière QoS                          | CME sur DIST-SW1 (Active + root du VLAN 30) ; héritage P3 vérifié — résumé `/20` audité sûr, aucune reconfig |
| [P6](./P6/README.md) | WiFi                 | WLC, APs lightweight, AP autonome               | CAPWAP, SSID Corp 301 / Guest 310, WPA2, HSRPv2 VLAN 300            | VLAN 300 consolidé sur DIST-SW1 (Active + root + DHCP + WLC) ; contrôle et data plane prouvés séparément     |

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

## Écarts de production

| Écart                      | État actuel (lab, as-built)                                                                                                        | Direction production                                                 |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Authentification OSPF      | Non implémentée                                                                                                                    | Authentification des adjacences sur les liens routés                 |
| Plan de management         | SSH/AAA partiels ; pas d'IP de management dédiée (managé in-band, Loopback0 différé)                                               | SSHv2, AAA, ACL de management, journalisation centralisée, Loopback0 |
| Redondance DHCP            | Une autorité par domaine ; relais mono-chemin (helper côté Active seul), pool sur l'Active — pas de nouveau bail si l'Active tombe | Split-scope / HA, miroir sur le Standby                              |
| Routage ASA                | Résumés statiques inside (`/16` + `/20`) + verrous Null0 (AD 254) sur le HQ-Router ; audités sûrs (P5)                             | Routage dynamique (OSPF) sur le pare-feu                             |
| Data plane Wi-Fi           | Chemin client CAPWAP non simulé (prouvé par AP autonome)                                                                           | Vrai WLC, Cisco CML, GNS3 ou lab physique                            |
| Isolation Guest (VLAN 310) | Configurée, non prouvée en data plane (pas de portail captif PT)                                                                   | Guest anchor WLC + portail captif                                    |
| Load balancer datacenter   | VIP présentée, round-robin conceptuel (pas de moteur LB en PT)                                                                     | LB fonctionnel (HAProxy / F5), health-checks                         |
| Contrôle VoIP              | Point CME/TFTP unique                                                                                                              | Contrôle d'appel + services TFTP redondants (Loopback)               |
| Passerelle datacenter      | Un SVI par Leaf (SPOF par tier)                                                                                                    | Anycast Gateway via VXLAN EVPN                                       |
|                            |                                                                                                                                    |                                                                      |

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
