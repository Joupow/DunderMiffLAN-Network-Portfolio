# TheBigOffice : Vue d'ensemble technique  

## Objectif

Ce document donne à un relecteur technique une vue **consolidée** du portfolio Packet Tracer TheBigOffice : architecture, choix de conception, limites connues du lab, apprentissages. Il porte le consolidé unique des **écarts de production**.

À lire comme un **lab d'ingénierie réseau junior, pas comme une architecture de référence de production**. 

L'état décrit est l'état **as-built** : chaque partie (P1→P6) livre une base propre validée avant la suivante, et les décisions structurantes sont explicitées au fil du texte. Le détail de reproduction vit dans les `WORKFLOW_PX` ; l'adressage dans `IPAM.md` ; ce document renvoie, il ne recopie pas.

---
## Ce qu'un reviewer challengera

**« Pourquoi Packet Tracer ? »** 

PT sert à pratiquer et documenter des concepts de niveau Network+ dans une topologie simulée cohérente. Il démontre la logique de configuration, mais ne modélise pas les fonctionnalités avancées. Le projet s'arrête à la Partie 6 parce qu'au-delà, chaque sujet demanderait GNS3 / Cisco CML ou de l'équipement physique pour valider un comportement réel. Cette limite est **documentée, pas cachée**.

**« Est-ce prêt pour la production ? »** 

Non, et les écarts sont suivis explicitement (table ci-dessous). C'est un lab de portfolio junior ; l'honnêteté sur les limites est le levier de crédibilité.

**« Qu'est-ce qui prouve que ce n'est pas du copier-coller ? »** 

Trois choses qu'un copier-coller ne produit pas : 

- Le raisonnement de conception est **justifié**, pas seulement appliqué : pourquoi HSRP sur la Distribution et non le Core, pourquoi un `/20` transit contigu résumable en une route à l'ASA, pourquoi les deux Border Leafs sur le Core (ECMP) ; 

- Les **incidents de build sont documentés** comme apprentissages (table anti-copier-coller) ; 

- Les **matrices de validation sont honnêtes** — chaque partie distingue ✅ prouvé / ⚠️ configuré non prouvé / ❌ non simulable, et un flux bloqué se prouve par le **compteur d'ACL**, jamais par un timeout.

---

## Intention d'architecture

Le lab simule un petit réseau d'entreprise en six blocs fonctionnels : 

- (1) fondations LAN du siège, 
- (2) routage et redondance, 
- (3) pare-feu périmétrique et DMZ, 
- (4) fabric datacenter, 
- (5) services VoIP, 
- (6) services Wi-Fi. 

L'objectif est de démontrer comment **commutation, routage, sécurité, services, haute disponibilité et dépannage** interagissent dans un environnement cohérent unique. Chaque bloc s'appuyant sur une base validée par le précédent.

---

## Résumé technique par partie

| # | Bloc | Équipements principaux | Concepts principaux | Point notable |
|---|---|---|---|---|
| P1 | LAN siège | Core 3650, 2× Distribution 3560, 4× Access 2960 | VLANs, trunks, **Rapid PVST+**, VLAN de management, natif 999 trou noir | Roots STP sur la Distribution, alignés sur le plan HSRP de P2 ; aucun Access n'est root |
| P2 | Routage & redondance | Core, Distribution, HQ-Router | HSRP réparti, OSPF `/30`, DHCP par domaine + relais unique | Relais DHCP pointant le **HQ-Router (`10.0.1.2`)**, pas le Core ; bascule Core-first pour éviter un `.1` en double |
| P3 | Périmètre | ASA 5506-X, routeur FAI, services DMZ | Security-levels, 3 ACL opposées, NAT/PAT, IDS/SPAN, port-security | Route par défaut **originée dans OSPF** ; résumés internes verrouillés par **Null0 AD 254** |
| P4 | Datacenter | Spines / Leafs / **Border Leafs — tous 3650** | Spine-Leaf routée, fabric OSPF, tiers app + stockage, VIP LB | Les deux Border Leafs sur le Core (`10.0.12/13`, ECMP) ; objet PAT `DC-NET` sur l'ASA |
| P5 | Voix | CME, postes IP, switches d'accès | DHCP Option 150, SCCP, TFTP, frontière QoS | **CME co-localisé avec l'Active + root du VLAN 30** ; autorité DHCP = CME seul ; résumé `/20` audité sûr |
| P6 | Wi-Fi | WLC, APs lightweight, AP autonome | CAPWAP, SSID Corp 301 / Guest 310, WPA2, HSRPv2 VLAN 300 | VLAN 300 consolidé sur **DIST-SW1** (Active + root + DHCP + WLC) ; contrôle et data plane prouvés séparément |

---

## Adressage 

Le plan d'adressage complet : VLAN/zones, liens de transit `/30`, Router-IDs, autorité DHCP par domaine, conventions d'allocation est tenu en **source unique dans [`IPAM.md`](./IPAM.md)** 

En résumé : 

- L'espace utilisateur/voix/Wi-Fi est en `192.168.0.0/16`
- La DMZ et le datacenter en `172.16.0.0/16`
- Le **transit interne est un bloc contigu `10.0.0.0/20`** découpé en `/30` point-à-point (résumable en une seule route à l'ASA), et le périmètre utilise des plages publiques de test (RFC 5737). 
- Trois autorités DHCP, une par domaine de broadcast : HQ-Router (VLAN 10/20), CME (VLAN 30), DIST-SW1 (VLAN 300).

---

## Revue routage & IPAM

Le modèle de routage principal : **liens de transit point-à-point `/30`** et **OSPF aire 0**. La conception **évolue du plus simple au plus robuste** :

- **P1** simplifie volontairement le routage inter-VLAN sur le Core (SPOF documenté, temporaire).

- **P2** le durcit : SVIs et VIP HSRP sur la Distribution (réparties), liens Core-Distribution en `/30` routés, OSPF sur ces liens (SVIs LAN annoncés passifs), Core réduit au transit L3 pur. Le relais DHCP vise le **HQ-Router (`10.0.1.2`)** directement, pointer le Core créerait un double relais.

- **P3** ajoute le chemin de retour Internet : route par défaut statique sur le HQ-Router, **originée dans OSPF** (`default-information originate`). L'intérieur de l'ASA est décrit par des **résumés** (`192.168.0.0/16`, `10.0.0.0/20`) plutôt que par une multitude de `/30`.

- **P4** ajoute la fabric derrière deux Border Leafs sur le Core. L'espace transit étant un `/20` contigu, la fabric entre dans le résumé **sans renumérotation**.

- **P3/P5** traitent le risque de boucle à l'ASA. 

La leçon : **un résumé est dangereux dès qu'il inclut de l'espace non alloué.** Plutôt qu'énumérer chaque `/30`, la conception garde les résumés compacts et pose des **routes de rejet Null0 (AD 254)** sur le HQ-Router comme filet ; le LPM garantit qu'une vraie route OSPF gagne toujours, Null0 ne s'active que pour l'inconnu. P5 audite le `/20` et confirme qu'il ne résout que vers de l'espace alloué.

---

## Revue de conception par domaine

**Commutation.** 

- Segmentation VLAN et séparation access/trunk ; 
- **Rapid PVST+ sur chaque switch** (un seul laissé en PVST classique fait retomber le segment sur les timers 802.1D) ; 
- Root STP fixé sur la Distribution, réparti pour correspondre au plan Active/Standby HSRP ; 
- Durcissement du VLAN natif (nettoyage du 1, natif 999 trou noir) ; 
- PortFast + BPDU Guard indissociables sur les ports vers postes / téléphones / APs ; 
- Voice VLAN sur un seul câble (data untagged + voix taggée).

**Sécurité.** 

- L'ASA raisonne en **niveaux de confiance** (outside 0 / dmz 50 / inside 100), pas en ports : supérieur→inférieur passe, inférieur→supérieur est bloqué par défaut. 
- Toute la politique est l'ensemble des exceptions, exprimées en **trois ACL aux philosophies opposées** — le `permit ip any any` final est obligatoire sur inside et interdit sur DMZ (détail en README/WORKFLOW P3). 
- Chaque règle est prouvée par son **compteur de hits**. S'y ajoutent port-security sticky, IDS passif via SPAN, et NAT/PAT (deux objets PAT dynamiques + une publication statique, plus `DC-NET` en P4).

**Voix.** 

- La chaîne de boot d'un poste est prouvée par l'état d'appel : VLAN voix → DHCP (CME) → **Option 150** vers TFTP/CME (`.254`) → téléchargement → enregistrement **SCCP** → appel `Connected`. 
- La QoS introduit une **frontière de confiance** : DSCP EF cru seulement pour un téléphone Cisco identifié par CDP (anti-usurpation).

**Wi-Fi.** 

- Validation du **plan de contrôle séparée de celle du plan de données** (PT ne simule pas le data plane client CAPWAP) : le WLC prouve la gestion à base de contrôleur (4 APs `Online`) et le provisioning SSID, un **AP autonome** prouve le trafic client sans-fil→filaire (TTL 127). 
- Le VLAN 300 est consolidé sur DIST-SW1, aligné sur l'Active/root du VLAN 30 pour que le build Wi-Fi ne perturbe jamais la co-localisation CME de P5.

---

## Décisions de conception importantes

| Décision | Domaine | Pourquoi elle compte |
|---|---|---|
| Passerelles HSRP réparties sur la Distribution | Routage / HA | Supprime le SPOF de passerelle et étale la charge (les deux VLANs lourds sur des switches différents) |
| Router-ID OSPF codés en dur | Routage | Évite un comportement d'adjacence instable |
| OSPF tenu hors du VLAN de management | Routage / sécurité | Réduit l'exposition du plan de contrôle |
| Transit planifié en un `/20` contigu | Routage / IPAM | Une seule route de résumé à l'ASA ; la fabric entre sans renumérotation |
| Résumés ASA adossés à des rejets Null0 (AD 254) | Routage / sécurité | Neutralise la boucle vers l'espace non alloué sans énumérer chaque `/30` |
| Deux Border Leafs sur le Core (fabric tout en 3650) | Datacenter | Sépare N-S de E-O et fournit l'ECMP ; le 3650 apporte 4 downlinks par Spine |
| Une autorité DHCP par domaine de broadcast | Services | Supprime les réponses DHCP dupliquées ou ambiguës |
| Séparation plan de contrôle / plan de données au WLC | Wi-Fi | Évite de revendiquer une validation que PT ne peut pas prouver |

---

## Écarts de production (le consolidé d'honnêteté)

> Source unique de la table « écarts prod » du dépôt. Chaque écart est un choix de périmètre lab assumé, avec sa direction de durcissement.

| Écart | État actuel (lab) | Direction production |
|---|---|---|
| Authentification OSPF | Non implémentée | Authentification des adjacences sur les liens routés |
| Plan de management | SSH/AAA partiels | SSHv2, AAA, ACL de management, journalisation centralisée |
| Redondance DHCP | Une autorité par domaine, sans secours | Infrastructure DHCP dédiée, split-scope / HA |
| Routage ASA | Résumés statiques internes + filet Null0 (AD 254) | Routage dynamique (OSPF) sur le pare-feu |
| Data plane Wi-Fi | Chemin client CAPWAP non simulé (prouvé par AP autonome) | Vrai WLC, Cisco CML, GNS3 ou lab physique |
| Isolation Guest (VLAN 310) | Configurée, non prouvée en data plane (pas de portail captif PT) | Guest anchor WLC + portail captif |
| Contrôle VoIP | Point CME/TFTP unique | Contrôle d'appel + services TFTP redondants (Loopback) |
| Passerelle datacenter | Un SVI par Leaf (SPOF par tier) | Anycast Gateway via VXLAN EVPN |

---

## Preuve anti-copier-coller (incidents documentés)

| Point | Impact | Diagnostic / conception retenue |
|---|---|---|
| Collision de Router-ID OSPF | Instabilité d'adjacence possible | Router-ID **codés en dur** et documentés |
| Résumé ASA vers de l'espace non alloué | Boucle possible ASA ↔ Core | Résumés compacts adossés à des **routes Null0 (AD 254)** : le *longest-prefix match* fait toujours gagner une vraie route OSPF ; Null0 n'attrape que l'inconnu |
| Réponses DHCP dupliquées possibles | Conflits de baux | **Une autorité DHCP par domaine de broadcast**, prouvée par l'**absence** de tout second serveur sur le segment |
| SCCP frappe un switch sans service voix | Poste jamais enregistré | `ip source-address` sur l'IP réelle du CME (`.254`), jamais la VIP ; liaison `button` explicite poste→DN |
| Data plane client WLC non prouvable en PT | Forwarding CAPWAP invalidable | Plans de contrôle et de données validés **séparément** (contournement par AP autonome) |

