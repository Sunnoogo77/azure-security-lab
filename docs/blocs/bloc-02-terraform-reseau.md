# Bloc 2 — Terraform et réseau

> **État : en cours.**
> Partie 1 — backend de state distant : **terminée**.
> Partie 2 — réseau hub-and-spoke, NSG deny-by-default, Azure Policy : à venir.

---

## 1. À quoi sert ce bloc

Le bloc 1 a été construit entièrement à la main, en Azure CLI. C'était volontaire : on ne peut pas automatiser ce qu'on n'a jamais fait soi-même. Mais une infrastructure créée à la main a deux défauts rédhibitoires en entreprise :

- **elle n'est pas reproductible** — personne ne peut la recréer à l'identique, pas même son auteur six mois plus tard ;
- **elle n'est pas auditable** — il n'existe aucun document qui dise ce qui *devrait* exister, donc aucun moyen de détecter une dérive.

Le bloc 2 remplace les commandes par du code. À partir d'ici, toute ressource du lab est décrite dans un fichier Terraform, et toute modification passe par ce fichier.

Cette première partie ne déploie aucune ressource d'infrastructure. Elle construit l'endroit où Terraform gardera sa mémoire — sans quoi tout le reste est impossible.

---

## 2. Le state : pourquoi il ne peut pas rester en local

Terraform compare l'état décrit par le code avec l'état réel d'Azure, puis applique la différence. Pour faire cette comparaison, il doit savoir **quelles ressources il a créées lui-même**. Il tient donc un inventaire : le *state*.

Le state n'est pas un journal, c'est une mémoire : il décrit ce qui existe maintenant, pas ce qui s'est passé.

Sans state, Terraform ne distingue pas ses propres ressources de celles qui existaient déjà dans l'abonnement — ici, celles créées pendant la préparation de l'AZ-104. Un Terraform amnésique est un Terraform qui veut tout recréer.

Par défaut, ce state est un fichier local, à côté du code. Quatre raisons de le déplacer :

| Risque | Conséquence concrète |
|---|---|
| **Le state contient des secrets en clair** | Terraform y recopie les clés et mots de passe générés par Azure, même s'ils n'apparaissent nulle part dans le code. Le dépôt étant public, un state versionné serait une fuite directe — RGPD art. 32 (sécurité du traitement) et art. 5(1)(f) (confidentialité) |
| **Perte du fichier** | Terraform se croit face à un abonnement vide, tente de tout recréer, Azure refuse les noms déjà pris. L'infrastructure devient ingérable : ni modifiable, ni destructible proprement |
| **Pas de verrouillage** | Deux exécutions simultanées écrivent l'une sur l'autre. Sur un backend Azure Blob, Terraform pose un *lease* : la deuxième exécution attend au lieu de corrompre |
| **CI/CD impossible** | Au bloc 4, Terraform sera exécuté par un runner GitHub Actions éphémère, qui n'a pas accès au poste de travail. Le state doit être dans un emplacement partagé |

Le state est donc traité dans ce projet comme **une donnée sensible à part entière**, pas comme un fichier de travail.

---

## 3. Le paradoxe de l'amorçage

Le state doit vivre dans un Storage Account. Mais ce Storage Account ne peut pas être créé par Terraform : il faudrait déjà un state pour s'en souvenir, et le state n'existe pas encore.

Trois réponses possibles :

| Approche | Qui crée le backend | Compromis |
|---|---|---|
| **CLI manuelle, documentée** *(retenue ici)* | Azure CLI, une seule fois | Simple et explicite ; la ressource n'est pas gérée par Terraform et doit être documentée comme telle |
| Terraform avec state local puis migration | Terraform lui-même | Reproductible, mais le state local initial transite par le poste de travail |
| Provisionnement par la plateforme | Équipe plateforme / Landing Zone | Standard en entreprise, suppose une organisation qui n'existe pas dans un lab |

**Décision** : création en CLI, et mention explicite dans la documentation que `rg-asl-tfstate` n'est pas sous gestion Terraform. C'est la seule ressource permanente du projet.

**Coût** : Standard_LRS, quelques dizaines de kilo-octets → moins de 0,05 €/mois. Suppression : `az group delete -n rg-asl-tfstate`.

---

## 4. Mise en œuvre

### 4.1 Le resource group

```console
$ az group create \
    --name rg-asl-tfstate \
    --location francecentral \
    --tags project=azure-security-lab environment=shared owner=caleb cost-center=lab

{
  "id": "/subscriptions/<subscription-id>/resourceGroups/rg-asl-tfstate",
  "location": "francecentral",
  "managedBy": null,
  "name": "rg-asl-tfstate",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": {
    "cost-center": "lab",
    "environment": "shared",
    "owner": "caleb",
    "project": "azure-security-lab"
  },
  "type": "Microsoft.Resources/resourceGroups"
}
```

`environment=shared` et non `dev` : ce resource group sert tous les environnements. C'est le seul du projet dans ce cas.

### 4.2 Un nom imprévisible

Le nom d'un Storage Account est unique à l'échelle mondiale et se résout publiquement en `<nom>.blob.core.windows.net`. Un nom déductible — `aslprodtfstate` — s'énumère.

```console
$ SUF=$(openssl rand -hex 3)
$ echo "stasltfstate$SUF"
stasltfstate<suffixe>
```

Ce n'est pas une mesure de sécurité : c'est une réduction de surface. La serrure doit tenir sans elle, et c'est l'objet du point suivant.

### 4.3 Le Storage Account, verrouillé à la création

```console
$ az storage account create \
    --name stasltfstate$SUF \
    --resource-group rg-asl-tfstate \
    --location francecentral \
    --sku Standard_LRS \
    --kind StorageV2 \
    --min-tls-version TLS1_2 \
    --allow-blob-public-access false \
    --allow-shared-key-access false \
    --https-only true \
    --tags project=azure-security-lab environment=shared owner=caleb cost-center=lab \
    -o none && echo "storage créé"
storage créé
```

| Paramètre | Ce qu'il empêche |
|---|---|
| `--allow-blob-public-access false` | Qu'un blob devienne lisible anonymement, par erreur ou par malveillance. C'est le vecteur du scénario d'attaque M2 |
| `--allow-shared-key-access false` | Que les clés de compte servent à s'authentifier. **Le paramètre le plus important de la commande** — voir §5 |
| `--min-tls-version TLS1_2` | L'interception sur les suites cryptographiques obsolètes de TLS 1.0 / 1.1 |
| `--https-only true` | Le transit en clair |

Le chiffrement au repos ne figure pas dans cette liste : sur Azure Storage il est actif en permanence et ne peut pas être désactivé. Ce n'est pas une option, donc pas une décision.

### 4.4 Historique du state

```console
$ az storage account blob-service-properties update \
    --account-name stasltfstate$SUF \
    --resource-group rg-asl-tfstate \
    --enable-versioning true \
    --enable-delete-retention true \
    --delete-retention-days 7 \
    -o none && echo "historique activé"
historique activé
```

Terraform n'a pas besoin de ces deux options pour fonctionner. Elles sont là pour le parti pris du projet : **le versioning du state est l'historique de ce que l'infrastructure a été**. En analyse post-incident, c'est ce qui permet de répondre à « depuis quand cette règle était-elle ouverte ? » — une question à laquelle beaucoup d'organisations ne savent pas répondre.

Sept jours de rétention : assez pour une erreur humaine, assez court pour rester cohérent avec la minimisation (RGPD art. 5(1)(e)).

### 4.5 Le rôle sur le plan de données

```console
$ SA_ID=$(az storage account show \
    --name stasltfstate$SUF \
    --resource-group rg-asl-tfstate \
    --query id -o tsv)

$ az role assignment create \
    --assignee-object-id <mon-object-id> \
    --assignee-principal-type User \
    --role "Storage Blob Data Contributor" \
    --scope $SA_ID \
    -o none && echo "rôle attribué"
rôle attribué
```

Le triplet RBAC du bloc 1 s'applique :

| Terme | Valeur | Pourquoi |
|---|---|---|
| **QUI** | mon identité Entra ID | nominative, journalisée, révocable |
| **QUOI** | `Storage Blob Data Contributor` | lire et écrire le contenu des blobs, et rien d'autre |
| **OÙ** | le Storage Account seul | ni l'abonnement, ni le resource group : le moindre privilège vit dans la portée |

En entreprise, la portée descendrait jusqu'au container. Impossible ici : le container n'existe pas encore au moment de l'attribution.

### 4.6 Le container

```console
$ az storage container create \
    --name tfstate \
    --account-name stasltfstate$SUF \
    --auth-mode login \
    -o none && echo "container créé"
container créé
```

`--auth-mode login` n'est pas cosmétique. Sans ce paramètre, la CLI tente d'abord d'utiliser les clés de compte — désactivées à l'étape 4.3 — et échoue. **La commande ne passe que parce qu'une identité nommée la porte.** C'est la vérification en direct du durcissement.

---

## 5. Pourquoi désactiver les clés de compte change tout

Un Storage Account expose par défaut deux clés d'accès. Quiconque détient une de ces clés *est* le Storage Account :

- **pas d'identité** — les journaux enregistrent un accès par clé partagée, sans sujet. Impossible de savoir qui a lu quoi ;
- **pas de révocation ciblée** — on ne peut pas retirer l'accès à une seule personne, seulement régénérer la clé et casser l'accès de tout le monde ;
- **pas d'expiration** — une clé fuitée reste valable indéfiniment.

Avec `--allow-shared-key-access false`, le seul moyen d'ouvrir le state devient une identité Entra ID : un utilisateur nommé, ou l'identité fédérée du pipeline au bloc 4. Chaque lecture porte alors un nom dans les journaux.

**C'est le parti pris du projet — la traçabilité — appliqué à la mémoire de l'infrastructure elle-même.** Sans cela, le state serait le seul composant du lab dont les accès seraient anonymes.

### Plan de contrôle et plan de données

Être `Owner` sur l'abonnement permet de créer, modifier et supprimer le Storage Account. Cela ne permet pas de lire son contenu.

| Plan | Ce qu'il gouverne | Rôles typiques |
|---|---|---|
| **Contrôle** (ARM) | l'existence et la configuration de la ressource | `Owner`, `Contributor`, `Reader` |
| **Données** | le contenu de la ressource | `Storage Blob Data Owner/Contributor/Reader` |

Cette séparation est la raison pour laquelle les clés de compte étaient dangereuses : elles donnaient le plan de données en contournant entièrement RBAC. Elle explique aussi pourquoi `Owner` ne suffisait pas à créer le container.

---

## 6. La configuration côté code

### Azure Resource Manager, et les langues qu'on lui parle

Toute création de ressource passe par **ARM**, le plan de contrôle d'Azure. Portail, Azure CLI, templates ARM et Terraform ne sont que quatre façons de s'adresser au même point d'entrée.

| Outil | Ce qu'on écrit | Ce qu'ARM reçoit |
|---|---|---|
| Portail | des clics | un appel à l'API ARM |
| Azure CLI | des commandes | un appel à l'API ARM |
| Template ARM | du JSON | un appel à l'API ARM |
| Terraform | du HCL | un appel à l'API ARM |

« ARM » désigne le plan de contrôle, pas le JSON. Le JSON s'appelle un *template ARM* : un dialecte parmi d'autres. La confusion entre les deux est fréquente.

Terraform est retenu plutôt que les templates ARM pour trois raisons : il tient un state, il couvre d'autres fournisseurs que Azure avec la même grammaire, et son cycle `plan` / `apply` rend le changement lisible avant d'être appliqué.

### Trois fichiers, et pourquoi trois

```
infra/environments/dev/
├── providers.tf          versionné    — version, provider, « le state est dans Azure »
├── backend.hcl.example   versionné    — la forme de l'adresse du state
└── backend.hcl           ignoré       — la valeur réelle
```

`providers.tf` :

```hcl
terraform {
  required_version = ">=1.9"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }

  backend "azurerm" {
    use_azuread_auth = true
  }
}

provider "azurerm" {
  features {}
}
```

- `required_version` garantit qu'une version ancienne de Terraform ne produira pas un résultat différent en silence.
- `~> 4.0` accepte les versions correctives, refuse la majeure suivante : on autorise les corrections, pas les ruptures.
- `use_azuread_auth = true` est **obligatoire** ici. Les clés de compte étant désactivées, Terraform doit s'authentifier par identité Entra ID ; sans cette ligne il cherche une clé et refuse de démarrer.
- Le bloc `backend` ne contient volontairement aucune coordonnée.

C'est une **configuration de backend partielle** : le code public déclare *où en général*, un fichier local fournit *où exactement*. Le dépôt étant public, le nom du Storage Account n'y figure pas — ce n'est pas un secret, mais on ne publie pas la cartographie de la cible. L'identifiant d'abonnement est fourni de la même manière, par la variable d'environnement `ARM_SUBSCRIPTION_ID`.

`backend.hcl.example` publie la forme et pas la valeur :

```hcl
resource_group_name  = "rg-asl-tfstate"
storage_account_name = "stasltfstate<suffixe>"
container_name       = "tfstate"
key                  = "dev.terraform.tfstate"
```

Le fichier d'exemple n'est pas décoratif : il est ce qui rend le dépôt reprenable par un tiers. Un dépôt sans exemple de configuration est un dépôt qu'on ne peut pas cloner utilement.

`key` préfixé par l'environnement prépare la séparation dev/prod : un state par environnement dans le même container, donc aucune exécution sur dev ne peut altérer la mémoire de prod.

### Initialisation

```console
$ export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
$ terraform init -backend-config=backend.hcl

Initializing the backend...

Successfully configured the backend "azurerm"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Finding hashicorp/azurerm versions matching "~> 4.0"...
- Installing hashicorp/azurerm v4.81.0...
- Installed hashicorp/azurerm v4.81.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!
```

`.terraform.lock.hcl` **est versionné** : il fige la version exacte du provider et son empreinte cryptographique. C'est la garantie que le pipeline du bloc 4 utilisera le même binaire que le poste de travail — une protection contre la compromission de chaîne d'approvisionnement, au même titre qu'un fichier de verrouillage de dépendances applicatives.

Le répertoire `.terraform/` est ignoré : il ne contient que des artefacts téléchargeables.

---

## 7. Erreurs rencontrées

Consignées parce qu'un lecteur qui refait ces étapes les rencontrera.

**Échec de résolution DNS pendant `az storage container create`.** La CLI renvoie une centaine de lignes de trace Python ; la seule ligne utile est :

```console
Failed to resolve 'login.microsoftonline.com' ([Errno 8] nodename nor servname provided, or not known)
```

Ce n'est ni une erreur d'autorisation ni une erreur Azure : la CLI n'a pas pu joindre le service d'authentification. Cause réelle : connexion par partage de réseau mobile, coupure DNS brève. Relancer suffit. L'enseignement opérationnel : **lire la première exception, pas la dernière** — la trace Python se lit de haut en bas, la cause est en haut.

**Confusion plan de contrôle / plan de données.** Première tentative de création du container en tant que `Owner`, sans rôle sur les données. Documenté en §5 : c'est l'erreur la plus instructive de la session.

---

## 8. Limites, et ce qu'on ferait ailleurs

| Sujet | Lab (ici) | PME | Entreprise |
|---|---|---|---|
| Accès réseau au state | public réseau activé, RBAC seul garde-fou | restriction par plages IP du site | `publicNetworkAccess=Disabled` + private endpoint + runner auto-hébergé |
| Portée du rôle | Storage Account | container | container, via PIM avec activation à la demande |
| Redondance | `Standard_LRS` | `Standard_ZRS` | `Standard_GZRS` + verrou de suppression |
| Création du backend | CLI manuelle documentée | module de bootstrap versionné | fourni par l'équipe plateforme (Landing Zone CAF) |
| Isolation dev/prod | un state par environnement, même compte | un Storage Account par environnement | un abonnement par environnement |

La limite la plus sérieuse est la première : le state est joignable depuis internet, et seule l'authentification Entra ID le protège. En production, un private endpoint supprimerait cette exposition — mais imposerait que les exécutions Terraform partent d'un réseau connecté au VNet, donc un runner auto-hébergé. Ce compromis est assumé et documenté, pas ignoré.

**Reste à corriger dans ce bloc** : l'alerte budgétaire du bloc 1 a été créée au niveau du compte de facturation. Elle sera recréée au niveau de l'abonnement via `azurerm_consumption_budget_subscription`, pour être reproductible par le code.

---

## 9. Ce que je dois savoir défendre seul

1. **À quoi sert le state, et pourquoi il ne peut pas rester en local.** Mémoire de Terraform ; contient des secrets en clair ; sans lui, Terraform ne distingue pas ses ressources des autres ; inutilisable en CI/CD.
2. **Le paradoxe de l'amorçage et le choix fait.** Le backend ne peut pas être créé par Terraform ; créé en CLI, hors gestion Terraform, et documenté comme tel.
3. **Pourquoi `--allow-shared-key-access false` est la décision centrale.** Une clé partagée n'a ni identité, ni révocation ciblée, ni expiration. La désactiver rend chaque accès au state nominatif et journalisé.
4. **La différence entre plan de contrôle et plan de données.** `Owner` administre la ressource ; `Storage Blob Data Contributor` en lit le contenu. Les clés contournaient RBAC.
5. **Ce qu'est une configuration de backend partielle, et pourquoi elle est utilisée ici.** Le dépôt public déclare la nature du backend ; un fichier local non versionné fournit ses coordonnées.
6. **Pourquoi `.terraform.lock.hcl` est versionné alors que `.terraform/` ne l'est pas.** Le premier fige la version et l'empreinte du provider — intégrité de la chaîne d'approvisionnement. Le second n'est qu'un cache.
