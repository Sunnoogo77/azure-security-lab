# Scénarios d'attaque — pourquoi ce socle est construit ainsi

> **Note de lecture.** Les chapitres précédents énoncent des menaces sous forme de tableau. Un tableau se comprend, mais il ne se retient pas et il ne convainc pas une direction. Ce chapitre raconte donc deux attaques **de bout en bout**, telles qu'elles se produisent réellement, et montre à chaque étape la faiblesse exploitée, ce qui l'aurait arrêtée, et ce que l'incident a coûté.
>
> L'entreprise s'appelle **Novara Conseil** : cabinet de conseil B2B français, 200 collaborateurs, trois sites. **Caleb** vient d'y être recruté comme ingénieur cloud. Il n'est ni négligent ni incompétent — c'est important : aucune des erreurs racontées ici n'est stupide. Toutes ont été commises par des ingénieurs sérieux, pressés, et persuadés d'avoir fait le nécessaire.

---

## Scénario 1 — Le classeur resté sur le trottoir

*(Menace M2 : exposition publique d'un service de données)*

### Le récit

Mardi, 17 h 40. Caleb migre le portail client de Novara vers Azure. Les livrables de mission — audits, rapports stratégiques, analyses financières de clients — doivent être déposés dans un Storage Account. L'application web les lit, les affiche, les laisse télécharger.

Le premier essai échoue : l'application reçoit un `403 Forbidden` en tentant de lire les documents. Caleb cherche dix minutes, trouve sur un forum une réponse qui revient souvent, et coche la case qu'elle recommande : **autoriser l'accès public anonyme aux blobs**. Il rechargera la page, et ça fonctionnera. Le portail s'affiche, les documents se téléchargent, la démonstration de jeudi est sauvée.

Caleb se note mentalement de « revenir là-dessus plus tard ». Tout le monde connaît ce « plus tard ». Il n'arrive jamais, parce qu'il n'est inscrit nulle part : ni dans un ticket, ni dans une revue de code, ni dans une alerte.

Le compte de stockage s'appelle `novaraconseilprod`. Caleb l'a nommé comme ça parce que c'était lisible.

Onze semaines passent. Personne n'est attaqué — rien ne se voit.

Pendant ce temps, à l'autre bout du monde, un script tourne. Il ne vise pas Novara : il ne vise personne. Il lit la liste publique des entreprises françaises, en dérive des noms plausibles de comptes de stockage — `novaraconseil`, `novara-prod`, `novaraconseilprod` — et interroge pour chacun l'adresse `https://<nom>.blob.core.windows.net`. Il n'y a rien à pirater : c'est l'adresse publique et documentée du service. La plupart des noms ne répondent rien. Celui-ci répond.

Le script demande alors la liste du conteneur. Elle lui est donnée, en XML, sans qu'aucune authentification ne soit demandée. Trois mille quatre cents fichiers PDF. Il les télécharge tous en quarante minutes.

Il n'y a pas eu d'intrusion. Aucun pare-feu n'a été franchi, aucun mot de passe cassé, aucune vulnérabilité exploitée. **Le service a fonctionné exactement comme il était configuré pour fonctionner.**

Novara l'apprendra cinq mois plus tard, par un journaliste qui a retrouvé l'un de ses rapports d'audit en vente sur un forum.

### La feuille de route de l'attaquant

| Étape | Ce que fait l'attaquant | Ce dont il a besoin | Ce que Novara voit |
|---|---|---|---|
| 1. Reconnaissance | Génère des noms de comptes plausibles à partir du nom de l'entreprise | Rien — le format `<nom>.blob.core.windows.net` est public et universel | Rien |
| 2. Découverte | Teste l'existence de chaque nom par une requête HTTP | Rien. Un nom qui existe répond différemment d'un nom qui n'existe pas | Rien |
| 3. Vérification de l'accès | Demande la liste du conteneur (`?restype=container&comp=list`) | Que `allowBlobPublicAccess` soit activé et le conteneur en mode public | Rien, sauf si les journaux de diagnostic du Storage sont activés — ils ne le sont pas par défaut |
| 4. Exfiltration | Télécharge les 3 400 fichiers | Une connexion Internet | Un pic de trafic sortant sur une facture que personne ne lit ligne à ligne |

### Les faiblesses exploitées

Trois, et aucune n'est une vulnérabilité au sens logiciel du terme. Ce sont des **choix de configuration** :

1. **`allowBlobPublicAccess` activé** au niveau du compte, puis conteneur passé en accès anonyme. C'est l'erreur centrale, et c'est la première cause de fuite de données dans le cloud, tous fournisseurs confondus.
2. **`publicNetworkAccess` laissé ouvert** : le compte accepte les connexions de n'importe quelle adresse sur Internet. Même sans accès anonyme, cela maintient la porte d'entrée accessible pour une attaque par jeton SAS volé ou par clé compromise.
3. **Nom de ressource prévisible.** Point mineur mais réel : `novaraconseilprod` est devinable. Un suffixe aléatoire n'est pas une mesure de sécurité — c'est de l'obscurité, pas de la protection — mais il retire l'entreprise du champ des attaques automatisées non ciblées, qui représentent l'écrasante majorité du bruit de fond.

À quoi s'ajoute la faiblesse qui transforme l'incident en catastrophe : **l'absence de journaux**.

### Ce qui aurait arrêté l'attaque, et où

| Barrière | À quelle étape elle agit | Bloc du projet |
|---|---|---|
| `allow_blob_public_access = false` sur le Storage | Étape 3 : la liste du conteneur est refusée | Bloc 3 |
| `public_network_access_enabled = false` + **private endpoint** | Étape 2-3 : le service n'a plus d'existence sur Internet, il ne répond plus du tout | Bloc 3 |
| **Azure Policy** interdisant la création d'un Storage avec accès public | Empêche la réintroduction de l'erreur par n'importe qui, y compris Caleb un mardi à 17 h 40 | Bloc 2 |
| **Checkov** en intégration continue | L'erreur est refusée au moment de la *pull request*, avant même d'exister dans Azure | Bloc 4 |
| **Journaux de diagnostic** vers Log Analytics | N'empêche rien, mais permet de savoir *quels* fichiers ont été lus, *quand*, et *depuis où* | Bloc 5 |

Noter la gradation : la première barrière corrige l'erreur, la deuxième la rend impossible, la troisième empêche qu'elle revienne, la quatrième la détecte avant le déploiement. C'est ça, la défense en profondeur — pas « plusieurs pare-feux », mais **plusieurs occasions d'arrêter la même erreur, à des moments différents**.

### Ce que ça coûte à Novara

L'erreur technique a pris trente secondes. Voici la facture.

- **Obligation légale.** Il s'agit de données personnelles (contacts clients, auteurs des rapports). **RGPD, article 33** : notification à la CNIL dans les **72 heures** après connaissance de la violation. **Article 34** : si le risque pour les personnes est élevé, chaque personne concernée doit être informée individuellement. **Article 83** : les sanctions peuvent atteindre 20 millions d'euros ou 4 % du chiffre d'affaires mondial — plafond théorique, mais les sanctions réelles de la CNIL contre des PME se comptent en dizaines de milliers d'euros, et elles sont **publiées**.
- **Contractuel.** Novara vend du conseil : ses contrats contiennent tous une clause de confidentialité. Chaque rapport divulgué est un manquement contractuel distinct, chacun ouvrant droit à résiliation et à pénalités.
- **Commercial.** Un cabinet de conseil ne vend pas des documents, il vend de la confiance. La sanction la plus lourde ne vient pas du régulateur : elle vient des appels d'offres perdus les deux années suivantes.
- **Et le coût propre à l'absence de journaux — celui que personne n'anticipe.** Sans journaux de diagnostic, Novara ne peut pas déterminer *quels* fichiers ont été lus. Elle ne peut donc pas restreindre sa notification aux clients réellement touchés. Elle doit **supposer le pire** : notifier **tous** ses clients, y compris les 80 % qui n'ont rien subi. Un incident qui concernait quarante clients devient une annonce faite à deux cents.

> **Le détail qui change tout.** Deux entreprises subissent exactement la même attaque. La première a ses journaux : elle notifie quarante clients, documente précisément le périmètre, et démontre au régulateur qu'elle maîtrise son système d'information. La seconde ne les a pas : elle notifie tout le monde, ne peut rien prouver, et se présente devant la CNIL en disant « nous ne savons pas ». Le préjudice technique est identique. Le préjudice réel n'a rien à voir.
>
> C'est exactement pour cette raison que ce projet traite la traçabilité comme une exigence de premier rang.

---

## Scénario 2 — La clé oubliée dans l'historique

*(Menace M4 : divulgation de secrets)*

### Le récit

Caleb automatise les déploiements. Pour que son pipeline puisse créer des ressources dans Azure, il lui faut une identité : il crée un *service principal* et récupère les quatre valeurs habituelles — identifiant de client, identifiant de tenant, identifiant d'abonnement, et **un secret client valable deux ans**.

Le temps de faire fonctionner le tout, il écrit un petit fichier `local.auto.tfvars` avec les quatre valeurs dedans. C'est provisoire. Il le supprimera.

Vendredi 18 h 10, il valide son travail et pousse son code : `git add .`, `git commit`, `git push`. Le dépôt est public — Caleb en est fier, c'est son portfolio.

**Quarante secondes plus tard**, le secret est entre des mains étrangères. Ce n'est pas une image : l'API publique de GitHub diffuse en temps réel le flux des commits publics, et des dizaines de robots s'y abonnent précisément pour y chercher des identifiants. Le délai médian entre la publication d'une clé et sa première utilisation se compte en minutes.

Le lundi matin, Caleb s'aperçoit de sa bêtise. Il réagit vite et bien, croit-il : il supprime le fichier, commite, pousse. Le fichier n'apparaît plus sur GitHub.

**Le secret, lui, est toujours là.** Git ne supprime pas : il enregistre. Le commit de vendredi existe toujours dans l'historique, avec son contenu intact, accessible à quiconque clone le dépôt et remonte d'un commit. Supprimer un fichier dans Git produit un nouveau commit qui dit « ce fichier n'existe plus désormais » — pas « ce fichier n'a jamais existé ».

Ce que fait l'attaquant est discret. Il ne chiffre rien, ne vole rien de visible. Il se connecte à Azure avec l'identité du pipeline — une identité qui a le droit de créer des ressources — et déploie des machines de calcul dans trois régions, pour miner de la cryptomonnaie. C'est une technique répertoriée : **MITRE ATT&CK T1496, Resource Hijacking**.

Il choisit soigneusement des régions que Novara n'utilise pas. Personne ne regarde là-bas.

Novara le découvre cinq semaines plus tard, à la facture : **31 000 euros** au lieu de 900.

### La feuille de route de l'attaquant

| Étape | Ce que fait l'attaquant | Ce dont il a besoin | Ce que Novara voit |
|---|---|---|---|
| 1. Collecte | S'abonne au flux public des commits GitHub, filtre sur les motifs de secrets Azure | Rien, le flux est public et gratuit | Rien |
| 2. Validation | Teste le secret contre Entra ID | Que le secret soit encore valide — il l'est deux ans | Une authentification réussie dans les journaux d'Entra ID… si quelqu'un les consulte |
| 3. Reconnaissance | Liste les abonnements et les droits de l'identité | Que l'identité ait des droits larges — elle est Contributor sur l'abonnement | Des appels de lecture, noyés dans le bruit |
| 4. Exploitation | Déploie des machines de calcul dans des régions inutilisées | Le droit de créer des ressources | Rien, jusqu'à la facture |
| 5. Persistance | Ajoute un second secret à l'identité, pour survivre à la rotation du premier | Le droit d'écrire sur l'inscription d'application | Rien |

L'étape 5 est celle qu'on oublie, et c'est la plus coûteuse : une entreprise qui révoque le secret divulgué en croyant être tirée d'affaire laisse l'attaquant en place.

### Les faiblesses exploitées

1. **Un secret statique existait.** C'est la faiblesse mère : un secret qui n'existe pas ne peut pas fuiter.
2. **Il avait une durée de vie de deux ans.** Un jeton d'une heure aurait été inexploitable.
3. **L'identité du pipeline était Contributor sur tout l'abonnement**, alors qu'elle n'avait besoin de droits que sur deux groupes de ressources. Violation directe du moindre privilège.
4. **Aucun garde-fou en intégration continue** : rien, dans la chaîne, n'inspectait le code à la recherche de secrets.
5. **Aucune alerte budgétaire.** Cinq semaines de dérive sans signal.

### Ce qui aurait arrêté l'attaque, et où

| Barrière | À quelle étape elle agit | Bloc du projet |
|---|---|---|
| `.gitignore` excluant `*.tfvars` | Étape 0 : le fichier n'est jamais mis en suivi | Fait dès le départ |
| **Gitleaks** en intégration continue | Étape 1 : la *pull request* est refusée, le secret n'atteint jamais la branche principale | Bloc 4 |
| **OIDC avec identité fédérée** | Supprime la cause racine : **il n'y a plus aucun secret à divulguer**, seulement des jetons d'une heure liés à un dépôt et à une branche précis | Bloc 4 |
| **RBAC au périmètre du groupe de ressources**, rôles séparés pour `plan` et `apply` | Étape 3-4 : même compromise, l'identité ne peut pas créer de machines ailleurs | Blocs 1 et 4 |
| **Alerte budgétaire** à 50 %, 80 %, 100 % | Étape 4 : détection en quelques jours au lieu de cinq semaines | Bloc 1 |
| **Azure Policy** restreignant les régions autorisées | Étape 4 : le déploiement dans une région non prévue est purement refusé | Bloc 2 |

> **Ce qu'il faut retenir de ce tableau.** Six barrières, et une seule supprime vraiment le problème : OIDC. Les cinq autres limitent les dégâts d'un secret divulgué ; OIDC fait qu'il n'y a pas de secret. En sécurité, on appelle ça **éliminer la classe de vulnérabilité** plutôt que traiter l'instance. C'est toujours ce qu'il faut chercher en premier — et c'est rarement ce qu'on fait, parce que c'est ce qui demande de changer l'architecture plutôt que d'ajouter un contrôle.

### Ce que ça coûte à Novara

- **31 000 euros** de facture directe, non provisionnés, à justifier devant une direction financière.
- **Le doute, qui coûte plus cher que la facture.** L'identité compromise avait le droit de *tout lire* dans l'abonnement. A-t-elle seulement miné, ou a-t-elle aussi consulté le Key Vault et le Storage ? Sans journaux d'accès aux données, la réponse est : **on ne sait pas**. Et quand on ne sait pas, le droit impose de considérer qu'une violation de données personnelles a pu avoir lieu. Retour à l'article 33, et à la notification.
- **L'arrêt du déploiement**, le temps de refaire la chaîne d'authentification.

Deux scénarios, deux causes techniques sans rapport l'une avec l'autre. Et pourtant, dans les deux cas, la même phrase revient à la fin, et c'est elle qui coûte le plus cher : *on ne sait pas ce qui a été consulté.*
