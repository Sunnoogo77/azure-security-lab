# Azure Enterprise Security Lab

Socle Azure sécurisé pour une PME française simulée (200 collaborateurs) qui migre son portail client vers le cloud.
Réseau hub-and-spoke, workload sans accès public, Terraform, pipeline DevSecOps en OIDC, détection Sentinel.

**🚧 Projet en construction — avancement détaillé dans [`PROGRESS.md`](PROGRESS.md).**

---

## Le parti pris

**On ne se contente pas d'empêcher l'intrusion. On se donne les moyens de la reconstituer.**

Empêcher l'accès est la partie du métier que tout le monde traite. Savoir, après coup, *qui a fait quoi, quand, depuis où*, est celle que tout le monde néglige — jusqu'au jour de l'incident, où l'absence de journaux rend l'investigation impossible. Une organisation attaquée qui ne peut pas répondre à « qu'est-ce qui a été consulté ? » ne peut ni mesurer son préjudice, ni notifier ses clients, ni tenir le délai de **72 heures** imposé par l'article 33 du RGPD. Elle subit deux fois.

Conséquences assumées :

- la **traçabilité** est traitée au même rang que la confidentialité — journalisation centralisée, Microsoft Sentinel, et un bloc entier consacré au traitement d'un incident réel ;
- la **disponibilité** est volontairement hors périmètre, pour des raisons de coût documentées. C'est le choix inverse de la plupart des projets, et il est délibéré : la disponibilité est déjà le réflexe de tout ingénieur d'exploitation. La traçabilité, elle, n'a de défenseur que l'ingénieur sécurité.

## Architecture cible

| Couche | Contenu |
|---|---|
| **Hub** `rg-asl-<env>-hub` | VNet hub, `snet-security`, `AzureFirewallSubnet` réservé, zones DNS privées |
| **Spoke applicatif** `rg-asl-<env>-app` | VNet peeré, `snet-app` (intégration App Service), `snet-pe` (private endpoints) |
| **Workload** | App Service B1 Linux, Storage et Key Vault **sans accès public**, Managed Identity |
| **Sécurité** `rg-asl-shared-security` | Log Analytics, Microsoft Sentinel, Defender for Cloud |
| **CI/CD** | GitHub Actions, authentification **OIDC** — aucun secret statique |

Région : `francecentral`. Environnements éphémères : déployés, testés, prouvés, détruits.

## Avancement

| Bloc | Sujet | Statut |
|---|---|---|
| 0 | Cadrage, modèle de menaces, scénarios d'attaque | 🟢 Terminé |
| 1 | Compte et garde-fous : budget, Entra ID, RBAC | 🟢 Terminé |
| 2 | Terraform et réseau : backend, hub-and-spoke, NSG, Policy | 🟡 En cours |
| 3 | Workload privé : Key Vault, Storage, private endpoints | ⚪ À faire |
| 4 | Pipeline DevSecOps : OIDC, Checkov, Trivy, Gitleaks | ⚪ À faire |
| 5 | Détection : diagnostics, Sentinel, règles KQL | ⚪ À faire |
| 6 | Incident : escalade de privilèges, investigation, remédiation | ⚪ À faire |

## Documentation

| Document | Contenu |
|---|---|
| [`docs/00-cadrage.md`](docs/00-cadrage.md) | Contexte, données, conformité RGPD, modèle de menaces, périmètre |
| [`docs/01-scenarios-attaque.md`](docs/01-scenarios-attaque.md) | Deux attaques racontées de bout en bout, avec leur coût réel pour l'entreprise |
| [`docs/02-conventions-nommage.md`](docs/02-conventions-nommage.md) | Nommage, étiquetage, et les contraintes propres à Azure |
| [`docs/blocs/`](docs/blocs/) | Un rapport par bloc : raisonnement, mise en œuvre, preuves, limites |

Les rapports sont écrits pour être lus **sans connaissance préalable du projet** : chaque choix technique y est relié au risque métier qu'il traite.

## Hors périmètre, et pourquoi

| Exclu | Raison | Ce qu'on ferait en entreprise |
|---|---|---|
| Azure Firewall | ~900 €/mois | Inspection du trafic sortant, filtrage par FQDN, IDPS |
| Azure Bastion | ~130 €/mois | Accès administrateur sans IP publique |
| VPN / ExpressRoute Gateway | 25 à 300 €/mois | Raccordement des sites et du télétravail |
| Haute disponibilité | Choix de coût assumé | Multi-région actif/passif, RTO/RPO contractualisés |
| PIM (*Privileged Identity Management*) | Licence Entra ID P2 | Élévation de privilèges à la demande, bornée dans le temps |

Ces composants sont **documentés, pas déployés**. Savoir ce qu'on n'a pas fait, et pourquoi, fait partie du travail.
