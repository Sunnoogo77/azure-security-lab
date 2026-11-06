# Bloc 0 — Cadrage

> Ce document répond à une seule question : **pourquoi ce socle Azure est construit comme il l'est ?**
> Chaque choix technique des blocs 1 à 6 doit pouvoir être relié à une ligne de ce fichier.

---

## 0. Parti pris du projet

Ce socle est construit autour d'une conviction : **on ne se contente pas d'empêcher l'intrusion, on se donne les moyens de la reconstituer.**

Empêcher l'accès est la partie du métier que tout le monde traite. Savoir, après coup, *qui a fait quoi, quand, depuis où*, est la partie que tout le monde néglige — jusqu'au jour de l'incident, où l'absence de journaux rend l'investigation impossible. Une organisation attaquée qui ne peut pas répondre à « qu'est-ce qui a été consulté ? » ne peut ni mesurer son préjudice, ni notifier ses clients, ni respecter ses obligations légales. Elle subit deux fois.

Conséquences assumées dans ce projet :

- la **traçabilité** est traitée comme une exigence de premier rang, au même niveau que la confidentialité (Bloc 5, et Bloc 6 qui la met à l'épreuve sur un incident réel) ;
- la **disponibilité** est volontairement laissée de côté (coût, voir chapitre 7). C'est le choix inverse de la plupart des projets, et il est délibéré : la disponibilité est déjà le réflexe de tout ingénieur d'exploitation — personne n'est payé pour l'oublier. La traçabilité, elle, n'a de défenseur que l'ingénieur sécurité.

## 1. Contexte

- PME française de **conseil B2B**, **200 collaborateurs**, 3 sites et télétravail généralisé.
- Elle migre son **portail client** vers Azure. Aujourd'hui : un serveur auto-hébergé, sans sauvegarde testée ni supervision.
- Décision de l'entreprise : **hébergement des données en France** (exigence contractuelle de ses propres clients).

## 2. Le workload migré

Une application web interne (le portail client) qui :

- authentifie les collaborateurs,
- lit et écrit des **documents de mission** dans un stockage objet,
- consomme des **secrets** (chaînes de connexion, clés d'API partenaires).

Dans le lab, l'application est une image Docker open source : le code applicatif n'est **pas** le sujet. Le sujet est l'infrastructure qui l'entoure.

## 3. Données manipulées et sensibilité

| Donnée | Nature | Impact si elle fuite | Impact si elle est altérée |
|---|---|---|---|
| Contacts clients (nom, e-mail, téléphone) | Données personnelles | À COMPLÉTER | À COMPLÉTER |
| Contrats et propositions commerciales | Confidentiel entreprise | À COMPLÉTER | À COMPLÉTER |
| Livrables de mission | Confidentiel client | À COMPLÉTER | À COMPLÉTER |
| Données RH des 200 salariés | Données personnelles sensibles | À COMPLÉTER | À COMPLÉTER |
| Secrets techniques (clés, chaînes de connexion) | Identifiants | À COMPLÉTER | À COMPLÉTER |

## 4. Exigences de conformité

Une seule réglementation s'applique ici, mais elle s'applique vraiment : le **RGPD**.

| Article | Ce qu'il exige | Conséquence concrète dans ce projet |
|---|---|---|
| Art. 32 — Sécurité du traitement | Mesures techniques « appropriées » : chiffrement, confidentialité, intégrité, disponibilité, et **tests réguliers** | Chiffrement au repos et en transit, accès privé, et des preuves de test (captures, requêtes KQL) |
| Art. 33 — Notification de violation | Notifier la CNIL **sous 72 heures** après en avoir pris connaissance | Impossible sans journaux centralisés et conservés → justifie le Bloc 5 (LAW + Sentinel) |
| Art. 25 — Protection des données dès la conception et **par défaut** | « Par défaut, seules les données nécessaires… ne sont pas rendues accessibles à un nombre indéterminé de personnes » | C'est littéralement le **deny-by-default** : NSG en refus par défaut, accès public désactivé, Azure Policy |
| Art. 5 (1) (f) — Intégrité et confidentialité | Principe général de sécurité des données | **Moindre privilège** : RBAC par groupes, périmètre au groupe de ressources |
| Art. 5 (1) (e) — Limitation de la conservation | Ne pas conserver les données plus longtemps que nécessaire | Crée une **tension assumée** avec la traçabilité : une rétention de journaux trop courte rend l'investigation impossible, trop longue devient une collecte abusive. Choix à justifier (Bloc 5) |

Note : les données restent en France (`francecentral`). Ce n'est pas une obligation du RGPD en soi — le RGPD autorise l'UE entière — mais c'est une **exigence contractuelle** de l'entreprise. Savoir faire cette distinction est un point d'entretien.

## 5. Modèle de menaces

> À COMPLÉTER — c'est le cœur du cadrage.
> Méthode : pour chaque scénario, répondre à **qui ? par où ? pour obtenir quoi ? quel impact ?**

| # | Qui (attaquant) | Par où (vecteur) | Pour obtenir quoi | Impact métier |
|---|---|---|---|---|
| M1 | À COMPLÉTER | | | |

## 6. Exigences de sécurité dérivées

> À COMPLÉTER — une ligne par menace du chapitre 5. C'est ce tableau qui justifie chaque bloc.

| Menace | Exigence de sécurité | Mise en œuvre | Bloc |
|---|---|---|---|
| M1 | À COMPLÉTER | | |

## 7. Périmètre

**Dans le périmètre :** réseau hub-and-spoke, filtrage NSG, Azure Policy, identités et RBAC, Key Vault et Storage privés, Managed Identity, pipeline CI/CD avec OIDC et analyse statique, journalisation centralisée, détection Sentinel, traitement d'un incident.

**Hors périmètre, et pourquoi :**

| Exclu | Raison | Ce qu'on ferait en entreprise |
|---|---|---|
| Azure Firewall | ~900 €/mois, hors budget d'un lab | Déployé dans le hub, inspection du trafic sortant, DNS proxy |
| Azure Bastion | ~130 €/mois | Accès administrateur aux VM sans IP publique |
| VPN / ExpressRoute Gateway | ~25 à 300 €/mois | Raccordement du réseau des sites et du télétravail |
| Haute disponibilité (multi-région, sauvegardes) | Choix de coût assumé | Actif/passif multi-région, RTO/RPO contractualisés, tests de bascule |
| Sécurité du code applicatif | L'application est une image tierce | SAST/DAST sur le code, revue de dépendances |

## 8. Contraintes du projet

- **Une seule souscription** : séparation dev/prod par **resource groups + RBAC**, et non par souscription. En entreprise : CAF Landing Zone, management groups, une souscription par environnement.
- **Budget** : crédit gratuit de 200 USD, puis carte réellement débitée → alertes budget obligatoires (Bloc 1).
- **Environnements éphémères** : déployés, testés, prouvés, détruits.
- Région unique : `francecentral`.

## 9. Critères de succès

> À COMPLÉTER — comment sait-on que le socle est réussi ? Pense « preuve », pas « ressource déployée ».

## 10. Pitch de 2 minutes

> À COMPLÉTER PAR TOI, EN DERNIER. Le livrable réel du Bloc 0.
> Contrainte : 2 minutes à l'oral, sans notes, sans jargon inutile. Ordre conseillé :
> contexte → ce qui est en jeu → les 3 risques principaux → ce que le socle apporte → ce que j'ai volontairement exclu et pourquoi.
