# Réseau d'entreprise TheBigOffice
## Portfolio réseau junior — Cisco Packet Tracer / CompTIA Network+

Ce dépôt documente, en autodidacte, une architecture réseau d'entreprise complète : conception, configuration, validation, dépannage et gestion des limites — y compris les endroits où un choix initial était perfectible et comment il a été identifié puis corrigé.

L'objectif : construire de zéro un lab pratique couvrant l'ensemble du programme **CompTIA Network+** (et quelques sujets au-delà), sans s'arrêter à la théorie mais en ancrant chaque concept dans une implémentation qui fonctionne, jusqu'au débogage. **La certification Network+ a été obtenue** ; le lab en est la contrepartie pratique.

C'est un **lab d'apprentissage structuré, pas une architecture de référence de production** — conçu pour démontrer des compétences d'ingénierie réseau fondamentales : concevoir, configurer, dépanner et documenter un réseau d'entreprise cohérent. Le projet s'arrête volontairement à la Partie 6, là où Packet Tracer devient le facteur limitant : reconnaître qu'un outil a atteint sa limite et basculer vers GNS3 / Cisco CML est une décision d'ingénierie, pas un renoncement.

> **Ce projet est un rebuild.** La v1 était mon premier home lab autodidacte — trop d'erreurs techniques au regard de l'ambition, beaucoup de dépannage. Cette v2 vise l'efficacité : plus vite, plus propre, plus droit au but. Elle garde des imperfections, mais elle est presque reproductible telle quelle. La capacité à identifier et corriger ses propres erreurs est elle-même un livrable.

---

## Ce que ce portfolio démontre

| Domaine de compétence | Preuve dans le projet |
|---|---|
| Conception réseau | Lab connecté type entreprise : siège, périmètre, datacenter, voix, Wi-Fi |
| Commutation | VLANs, trunks 802.1Q, **Rapid PVST+ sur tous les switches**, natif 999 trou noir, PortFast + BPDU Guard |
| Routage | Inter-VLAN, OSPF aire 0 sur `/30` routés, origination du défaut, résumé + verrou Null0 |
| Haute disponibilité | Passerelles **HSRP sur la Distribution** (réparties Active/Standby), alignées sur les roots STP |
| Sécurité | ASA trois zones, DMZ, 3 ACL aux philosophies opposées, NAT/PAT, proxy forcé, port-security, SPAN/IDS |
| Services | Autorité DHCP par domaine + relais, NAT, VoIP/CME (Option 150, SCCP), TFTP, Wi-Fi à base de WLC |
| Dépannage | Collision de Router-ID, chaîne de boot VoIP, confinement de boucle à l'ASA, limites du simulateur WLC |
| Documentation | Chaque partie : notes de conception, CLI annotée, workflow, matrice de validation honnête, registre de dette |

---

## Périmètre du projet

| Partie | Sujet | Résultat principal |
|---|---|---|
| [P1](./P1/README.md) | Fondations LAN du siège | VLANs, trunks 802.1Q, VLAN de management, **Rapid PVST+ avec roots sur la Distribution** |
| [P2](./P2/README.md) | Routage & redondance | **HSRP sur la Distribution**, OSPF point-à-point, DHCP centralisé + relais unique, Core en transit L3 pur |
| [P3](./P3/README.md) | DMZ & pare-feu | ASA trois zones, NAT/PAT, 3 ACL, IDS/SPAN, `default-information originate` + verrou résumé/Null0 |
| [P4](./P4/README.md) | Datacenter | Spine-Leaf routée, **2 Border Leafs sur le Core (ECMP N-S)**, tiers app + stockage, VIP de load balancer |
| [P5](./P5/README.md) | VoIP | CME co-localisé avec l'Active/root du VLAN 30, DHCP Option 150, SCCP, TFTP, frontière QoS |
| [P6](./P6/README.md) | Wi-Fi | WLC + APs lightweight (CAPWAP), SSID WPA2 Corp/Guest, **HSRPv2 VLAN 300 sur DIST-SW1** |

> Vue consolidée (architecture, décisions transverses, écarts de production) → [`TECHNICAL_OVERVIEW.md`](./TECHNICAL_OVERVIEW.md). Schémas de topologie dans chaque partie.

---

## Principes transverses

Quelques règles de conception qui traversent tout le dépôt :

- **Prouver par un état ou un trafic réel, jamais par un écran de config.** Un flux bloqué se prouve par le compteur d'ACL, pas par un timeout.
- **Une seule autorité DHCP par domaine de broadcast** — pas un serveur unique pour tout (ce serait un SPOF), mais une autorité par segment (HQ-Router, CME, DIST-SW1). La preuve : **aucun** autre serveur ne répond.
- **Un résumé de routes est dangereux dès qu'il inclut de l'espace non alloué.** L'ASA décrit l'intérieur par des résumés compacts, sécurisés par des **routes Null0 (AD 254)** ; le *longest-prefix match* fait toujours gagner une vraie route OSPF.
- **La chaîne de boot VoIP casse silencieusement** (DHCP → TFTP → SCCP → enregistrement) : le symptôme apparaît souvent trois étapes après la cause.
- **Nommer les limites du simulateur, ne jamais les cacher.** Le data plane CAPWAP n'étant pas simulé, contrôle et données sont prouvés séparément — limite documentée, pas défaut caché.

---

## Carte du dépôt

```
README.md               # Vitrine : projet, compétences, périmètre, carte du dépôt
TECHNICAL_OVERVIEW.md   # Vue consolidée : architecture, écarts prod, décisions, apprentissages
IPAM.md                 # Plan d'adressage — SOURCE UNIQUE (VLAN/zones, /30, Router-IDs, autorité DHCP)
P1/ … P6/               # Une partie par dossier
  README.md             #   cadre : objectif, périmètre, compétences, matrice de validation
  WORKFLOW.md           #   reproduit : CLI annotée, validation par étape, registre de dette, annexe captures
assets/
  topologies/           # topology_pX.svg + topology_global.svg
  captures/  P1/ … P6/  # captures de preuve : Capture_PX_NN.png (référencées par [P-##])
  packet-tracer/        # fichiers .pkt du lab

# P1 LAN 3 niveaux · P2 HSRP/OSPF/DHCP · P3 ASA/DMZ/NAT · P4 Spine-Leaf · P5 VoIP/CME · P6 Wi-Fi/WLC
```

Pour chaque partie : le **README** cadre (objectif, périmètre, compétences, matrice de validation) ; le **WORKFLOW** reproduit (CLI annotée, validation par étape, registre de dette, annexe de captures).

---

## Usage de l'IA dans ce projet

L'assistance par IA a servi d'**outil d'apprentissage et de revue**, pas d'autorité ni de substitut à la compréhension du réseau : clarifier la syntaxe Cisco, challenger des choix de conception, relire des configs à la recherche d'erreurs courantes, structurer les prompts pour obtenir des réponses vérifiables.

L'essentiel était la **supervision**. Les sorties n'étaient pas tenues pour correctes par défaut : plusieurs points ont demandé revue et correction humaines — le risque de boucle du résumé ASA, l'ordre de la chaîne de boot VoIP, les limites du WLC en PT. Ce projet démontre donc aussi une compétence du travail technique réel : **piloter un assistant IA par un prompting clair et itératif, tout en gardant la responsabilité d'ingénierie et le jugement final du côté humain.**
