# PROGRESS — Azure Enterprise Security Lab

## Informations du compte

- Expiration du crédit gratuit : **12/11/2026** (6 jours restants au 06/11/2026)
- Crédit disponible au 06/11/2026 : **179,54 €** — coût engagé ce mois : 0 €
- Passage en paiement à l'usage : non
- Ressources permanentes actuellement déployées : aucune
- Garde-fou : budget `budget-asl-monthly` — 50 €/mois, alertes 50 % / 80 % réel et 100 % prévisionnel
- **Abonnement partagé** avec les labs du cours AZ-104 → inventaire avant tout déploiement, tags obligatoires, destruction en fin de session

> Les identifiants d'abonnement et de tenant ne figurent volontairement dans aucun fichier de ce dépôt public.

## Bloc en cours

**Bloc 2 — Terraform et réseau**

## Avancement

| Bloc | Statut | Date de fin | Rapport |
|---|---|---|---|
| 0. Cadrage | 🟢 Terminé | 06/11/2026 | [cadrage](docs/00-cadrage.md), [scénarios](docs/01-scenarios-attaque.md) |
| 1. Compte et garde-fous | 🟢 Terminé | 07/11/2026 | [bloc-01](docs/blocs/bloc-01-garde-fous.md) |
| 2. Terraform et réseau | 🟡 En cours | | |
| 3. Workload privé | ⚪ À faire | | |
| 4. Pipeline DevSecOps | ⚪ À faire | | |
| 5. Détection | ⚪ À faire | | |
| 6. Incident | ⚪ À faire | | |

## Journal des sessions

<!-- Une ligne par session : date — objectif — résultat — ce qui reste -->

- **06/11/2026** — Cadrage et scénarios d'attaque → `00-cadrage.md` et `01-scenarios-attaque.md` rédigés. Reste : chapitres 5, 6 et 9 du cadrage.
- **07/11/2026** — Bloc 1 : budget 50 €/mois + alertes, 3 groupes Entra ID, `grp-asl-security-readers` en Reader sur l'abonnement, convention de nommage. Bloc 1 terminé.

## Points à revoir / questions ouvertes

<!-- Ce que je n'ai pas encore complètement compris -->

- **PIM / JIT / ABAC** — notions citées au Bloc 1, à approfondir (licence Entra ID P2 absente, documentées sans être déployées).
- **Chapitres 5, 6 et 9 de `00-cadrage.md`** — modèle de menaces, exigences dérivées, critères de succès. À rédiger moi-même.
- **Budget au niveau compte de facturation** et non abonnement → en recréer un en Terraform au Bloc 2.
