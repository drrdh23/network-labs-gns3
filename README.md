# Network Labs (GNS3) — fondamentaux réseau et sécurité

> Série de labs progressifs réalisés sur **GNS3** avec routeurs Cisco c7200 : du câblage d'un bureau jusqu'au filtrage par ACL et au diagnostic méthodique de pannes.

![GNS3](https://img.shields.io/badge/GNS3-lab-2ea44f) ![Cisco IOS](https://img.shields.io/badge/Cisco-IOS%2015.2-1BA0D7) ![VPCS](https://img.shields.io/badge/hosts-VPCS-lightgrey)

*Labs réalisés dans le cadre du cours Networking Essentials (formation E1A, Geneva Institute of Technology). Les scénarios sont reformulés ; aucun énoncé de l'école n'est reproduit.*

| # | Lab | Thèmes | Dossier |
|---|---|---|---|
| 01 | Réseau d'un bureau de 8 postes | Adressage IPv4, switching, ARP | [`01-small-office-lan`](01-small-office-lan/) |
| 02 | Segmentation en VLAN | VLAN 802.1Q, isolation de domaines de diffusion | [`02-vlan-segmentation`](02-vlan-segmentation/) |
| 03 | Routeur pare-feu | NAT/PAT, ACL étendue, filtrage HTTP/Telnet | [`03-acl-firewall-nat`](03-acl-firewall-nat/) |
| 04 | Diagnostic de panne | Méthode OSI bottom-up, poste témoin, rapport d'intervention | [`04-troubleshooting-osi`](04-troubleshooting-osi/) |

Projets plus avancés dans des dépôts dédiés :
- **multisite-ipsec-vpn-lab** — VLAN + router-on-a-stick + VPN IPsec entre deux sites
- **ccna-lab-routing-nat-ssh** — VLSM, routage statique, NAT, SSH, intégration VMware

## Environnement commun

- GNS3, routeur Cisco c7200 (IOS 15.2), switch Ethernet intégré à GNS3, hôtes VPCS
- Nuage NAT GNS3 pour la sortie Internet (labs 03 et 04)
- Images Cisco IOS **non fournies** (licence propriétaire)

## Structure

```
network-labs-gns3/
├── 01-small-office-lan/
│   ├── README.md
│   └── configs/        # R1.cfg, PC1..PC8.vpc
├── 02-vlan-segmentation/
├── 03-acl-firewall-nat/
└── 04-troubleshooting-osi/
```

## Compétences démontrées

Adressage IPv4 · commutation et VLAN · NAT/PAT · listes de contrôle d'accès · méthodologie de dépannage · documentation technique

---
Auteure : **Rrudhani Deaa**
