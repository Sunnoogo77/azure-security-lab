# Conventions de nommage et d'étiquetage

> Une convention de nommage n'est pas de la cosmétique. C'est ce qui permet, six mois plus tard, de savoir ce qu'on regarde sans demander à personne — et, le jour d'un incident, de répondre en dix secondes à « est-ce que cette ressource est en production ? ».

## Règle générale

```
<type>-asl-<environnement>-<rôle>
```

- `<type>` : abréviation normalisée du type de ressource (voir table)
- `asl` : le projet — *azure-security-lab*
- `<environnement>` : `dev`, `prod`, ou `shared` pour ce qui est transverse
- `<rôle>` : la fonction de la ressource dans l'architecture

## Table des préfixes

| Type de ressource | Préfixe | Exemple |
|---|---|---|
| Resource group | `rg` | `rg-asl-dev-app` |
| Virtual network | `vnet` | `vnet-asl-dev-hub` |
| Subnet | `snet` | `snet-asl-dev-pe` |
| Network security group | `nsg` | `nsg-asl-dev-app` |
| Key Vault | `kv` | `kv-asl-dev-x7k2` |
| Storage account | `st` | `stasldevx7k2` |
| App Service | `app` | `app-asl-dev-portail` |
| Log Analytics workspace | `log` | `log-asl-shared-security` |
| Private endpoint | `pe` | `pe-asl-dev-kv` |
| Groupe Entra ID | `grp` | `grp-asl-security-readers` |

## Les deux exceptions, et pourquoi

**Storage accounts** : Azure impose 3 à 24 caractères, **minuscules et chiffres uniquement**, pas de tiret. D'où `stasldevx7k2` au lieu de `st-asl-dev-x7k2`.

**Noms globalement uniques** : Storage et Key Vault partagent leur espace de noms avec le monde entier (`<nom>.blob.core.windows.net`). Deux conséquences :

1. Un nom déjà pris fait échouer le déploiement → on ajoute un **suffixe aléatoire** de 4 caractères.
2. Un nom devinable expose à l'énumération automatisée décrite au [scénario 1](01-scenarios-attaque.md). Le suffixe n'est **pas** une mesure de sécurité — c'est de l'obscurité — mais il retire la ressource du champ des attaques non ciblées, qui sont l'essentiel du bruit de fond.

## Étiquettes obligatoires

Toute ressource porte ces quatre étiquettes, sans exception :

| Tag | Valeur | À quoi il sert |
|---|---|---|
| `project` | `azure-security-lab` | Isoler le lab des autres travaux sur l'abonnement partagé |
| `environment` | `dev` / `prod` / `shared` | Filtrer, et conditionner les politiques |
| `owner` | identifiant du responsable | Savoir qui appeler |
| `cost-center` | `lab` | Ventiler la facture |

Le tag `project` a une utilité immédiate dans ce projet : l'abonnement héberge aussi les travaux pratiques du cours AZ-104. Il permet de lister, et surtout de **détruire**, uniquement ce qui appartient au lab :

```bash
az resource list --tag project=azure-security-lab -o table
```

Au Bloc 2, une **Azure Policy** rendra ces étiquettes obligatoires : une ressource créée sans elles sera refusée. Une convention qu'on ne contrôle pas automatiquement est une convention que personne ne suit.

## Ce qu'on ferait en entreprise

Ce schéma s'inspire du **Cloud Adoption Framework** de Microsoft. En entreprise, on y ajoute généralement :

- un code de **région** (`frc` pour France Central) quand plusieurs régions coexistent ;
- un **numéro d'instance** (`-01`, `-02`) pour les ressources répliquées ;
- un code d'**unité organisationnelle** quand plusieurs entités partagent un tenant.

Ils sont omis ici : une seule région, une seule entité, pas d'instances multiples. **Ajouter un segment qu'on n'utilise jamais allonge les noms sans rien apporter.**
