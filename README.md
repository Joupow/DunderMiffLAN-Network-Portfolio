# TheBigOffice - Packet Tracer - Portfolio
## 📓 Home lab pratique pour la CompTIA Network+ (et au-delà)

Ce dépôt présente **TheBigOffice**, un laboratoire réseau construit de manière autodidacte pour mettre en pratique les compétences couvertes par la **CompTIA Network+**, tout en explorant des notions plus avancées. 

Ce laboratoire réseau simule une infrastructure d'entreprise sous Cisco Packet Tracer en six parties, construites de manière progressive, chacune venant enrichir et compléter la précédente.

| Partie               | Sujet                        | **Concepts clés**                                                                                   |
| -------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------- |
| [P1](./P1/README.md) | 🏗️ Fondations LAN 3 niveaux | Modèle hiérarchique Cisco 3 niveaux · VLANs · 802.1Q · STP · routage inter-VLAN · SVI · Rapid PVST+ |
| [P2](./P2/README.md) | 🧭 routage & redondance      | Routage · HSRP · DHCP · Relay Helper · OSPF P2P                                                     |
| [P3](./P3/README.md) | 🛡️DMZ & pare-feu            | ASA · DMZ · NAT/PAT · filtrage · ACL · IDS/SPAN                                                     |
| [P4](./P4/README.md) | 🖥️ Datacenter               | Spine-Leaf · Border Leafs · trafic E-O et N-S · ECMP · tiers serveurs · load balancer · stockage    |
| [P5](./P5/README.md) | 📞VoIP                       | VoIP · CME · TFTP · DHCP Option 150 · QoS voix                                                      |
| [P6](./P6/README.md) | 📶 WiFi                      | WLC · APs lightweight · CAPWAP · SSID WPA2 Corp/Guest · HSRPv2 VLAN 300                             |

Le projet n'a pas vocation à représenter une architecture de production clé en main. Il s'agit avant tout d'un **portfolio technique junior,** les erreurs de conception font partie du processus d'apprentissage et sont traitées comme des opportunités d'analyse et d'amélioration.

## Topologie du projet


![Topologie Global](./assets/topologies/topology-global.svg)

## Les choix techniques structurants

- **l'apprentissage d'une discipline de séquençage, pas seulement des protocoles configurés.** L'ordre de build a été important pour éviter les pannes _avant_ qu'elles existent : root STP posé avant la redondance (P1), Core routé et SVIs retirés avant de lever les VIP pour ne jamais provoquer de split-brain (P2), routage et NAT prouvés _sans ACL_ avant d'ajouter le filtrage (P3), fabric datacenter prouvée avant de brancher NAT (P4).

- **Un fil rouge unique : « le service suit l'Active ».** Par VLAN, Active HSRP = root STP = service hébergé sur le même boîtier. Principe posé en P1 et hérité jusqu'au CME (P5) et au Wi-Fi (P6). Chaque partie solde explicitement les dettes de la précédente : le lab se lit comme **une seule histoire d'ingénierie continue**, pas six exercices isolés.

- **Prouver par un état ou un trafic réel, jamais par un écran.** Un flux bloqué se prouve au compteur d'ACL, un poste par `REGISTERED in SCCP`, une autorité DHCP par le fait qu'_aucun autre serveur_ ne répond sur le segment, jamais par un timeout ni une capture « qui a l'air de marcher ».

- **Le débogage comme cœur de l'apprentissage.** Le motif récurrent du **chemin de retour** - l'OFFER DHCP qui ne sait pas revenir (P2), la route par défaut qu'il faut injecter dans OSPF (P3), plus collision de Router-ID et asymétrie TFTP : des pièges qui ne se comprennent qu'en les résolvant.

- **Nommer les limites de l'outil, jamais les cacher.** Le data plane CAPWAP n'étant pas simulé, contrôle (WLC, 4 APs `Online`) et données (AP autonome) sont prouvés séparément. C'est aussi ce qui justifie la clôture à P6.

- **IA en supervision, pas en autorité** :  L'IA a été utilisée comme un outil d'apprentissage, de relecture et d'aide à la conception, sans jamais remplacer la compréhension des concepts réseau.  Elle m'a permis de clarifier la syntaxe Cisco, d'identifier des incohérences et de vérifier les configurations tout au long de la réalisation du laboratoire. 

## Documentation

La documentation du laboratoire est structurée autour de ressources globales :

- 🏷️ [IPAM](./IPAM.md)
- 📘 [TECHNICAL_OVERVIEW](./TECHNICAL_OVERVIEW.md) 

Chaque partie du laboratoire dispose ensuite de son propre espace documentaire contenant : 

- 📄 **README**
- 🪜 **WORKFLOW** 
- 🧪 **Fichier Cisco Packet Tracer (.pkt)**

```
TheBigOffice - Packet Tracer Portfolio /
│
├── README.md               ← Vitrine : projet, compétences, périmètre, carte du dépôt
├── TECHNICAL_OVERVIEW.md   ← Architecture, prod, décisions, apprentissages  
├── IPAM.md                 ← Plan d'adressage, VLAN/zones, Router-IDs, autorité DHCP 
│
├── P1/ … P6/               ← Une partie par dossier
│   ├── README.md           ← Cadre : objectif, périmètre, compétences, matrice de validation   
│   ├── WORKFLOW.md         ← reproduit : CLI annotée, validation par étape, 
│   └── TBO-Part_X.pkt/     ← fichiers .pkt du lab
│ 
├── assets/
│   ├── topologies/           ← topology_pX.svg + topology_global.svg
│   ├── network-overview/     ← Captures topologie dans Packet Tracer : N0_pX.png
│   └── captures/  P1/ … P6/  ← captures de preuve : Capture_PX_NN.png 
└──
```

Cette organisation permet de suivre l'ensemble du cycle de vie d'une implémentation réseau :

- 🏛️ Conception de l'architecture réseau 
- 🧩 Segmentation et organisation logique du réseau
- ⚙️ Configuration des équipements 
- ✅ Validation du fonctionnement
- 🔎 Diagnostic et résolution d'incidents
- 📈 Analyse des limites de conception et amélioration de l'architecture

## Plan initial

Le plan initial incluait 11 parties avec d'avantages de sujets pour couvrir la majorité des objectifs de la CompTIA Network+ comme :  IPv6, monitoring, PKI/RADIUS, notions de cloud, simulation d'attaques. 

Le projet s'arrête à la Partie 6 parce que **Packet Tracer devient le facteur limitant**. 

Au-delà d'un certain niveau, contourner les limites du simulateur (CAPWAP, VXLAN, iSCSI, SNAT…) coûte plus de temps que produire de la configuration représentative.

Excellent outil d'apprentissage, Packet Tracer reste un simulateur : il modélise les protocoles sans exécuter un vrai IOS. 

**La suite se fera peut-être sur GNS3 / Cisco CML**, ou un autre logiciel de virtualisation, qui émulent de vraies images IOS et permettent de tester réellement ce que Packet Tracer ne peut que représenter, tout en continuant de monter en compétence sur des outils professionnels.

## 🏁 Conclusion


L'objectif initial de l'apprentissage est largement dépassé. 

Au-delà de la théorie CompTIA Network+, ce projet a imposé une pratique de terrain : concevoir une architecture cohérente, la mettre en œuvre et, surtout, la déboguer. 

Le débogage a produit l'apprentissage le plus réel : boucles de routage, conflits d'adressage, asymétries TFTP et collisions de Router-ID ne s'apprennent pas dans un cours ; ils se comprennent en les résolvant. 

Le pari de l'apprentissage par la pratique a payé, et il dépasse la simple préparation à la certification obtenue à ce jour avec succès. 
