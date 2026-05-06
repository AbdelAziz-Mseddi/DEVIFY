# Structure complète du cahier des charges — Parties V, VI et VII

---

## V- Modélisation des processus métier

Après avoir modélisé les interactions entre objets via les diagrammes de séquence, cette section adopte une perspective centrée sur les **processus** : plutôt que de montrer qui communique avec qui, on s'intéresse au **flux logique** des opérations, aux **décisions** prises à chaque étape et aux **parallélismes** possibles.

Les diagrammes d'activité UML permettent de représenter ces workflows sous forme de graphes d'activités, incluant des nœuds de décision, des boucles et des barres de synchronisation (fork/join). Ils sont particulièrement adaptés pour :
- détailler la **logique interne** d'un algorithme complexe,
- donner une **vue transversale** d'un processus impliquant plusieurs acteurs,
- modéliser les **scénarios alternatifs** avec leurs points d'échec.

Nous avons sélectionné trois processus complémentaires :

### 1) Algorithme d'attribution des Reviewers

Ce diagramme détaille la logique interne de la méthode `suggestReviewers()` du contrôleur `AttributionController`, qui constitue le cœur fonctionnel de la plateforme.

Dans le diagramme de séquence du cas *Revendiquer une revue d'une PR*, cet algorithme apparaissait comme un simple appel de méthode dans une boucle. Le diagramme d'activité permet d'en expliciter le fonctionnement étape par étape : récupération parallèle des données de chaque reviewer (charge de travail, historique, points, rating), calcul du score pondéré à l'aide des paramètres configurables de `AlgorithmParameters`, tri et sélection des 5 meilleurs candidats.

*[Insérer le diagramme d'activité de l'algorithme d'attribution]*

### 2) Cycle de vie complet d'une Pull Request

Ce diagramme offre une vue d'ensemble du parcours d'une Pull Request depuis sa création sur le dépôt externe jusqu'à sa fermeture. Il utilise des **swimlanes** pour distinguer les responsabilités de chaque acteur (Développeur, Système, Reviewer, Dépôt externe) et met en évidence les transitions entre les statuts définis par l'énumération `PullRequestStatus` (open → approved / rejected) ainsi que ceux de `ReviewStatus` (pendingReview → acceptedForReview → completedReview).

Ce diagramme est complémentaire aux diagrammes de séquence : là où chaque diagramme de séquence couvre un cas d'utilisation isolé, celui-ci montre comment ces cas s'enchaînent dans un scénario de bout en bout.

*[Insérer le diagramme d'activité du cycle de vie d'une PR]*

### 3) Processus d'authentification (Login / SignUp)

Ce diagramme modélise les deux parcours possibles lors de l'entrée dans le système : la connexion avec un compte existant (Login) et la création d'un nouveau compte (SignUp). Il met en évidence la boucle d'ajout de comptes externes (GitHub, GitLab, Bitbucket) via OAuth, ainsi que les trois points d'échec possibles (email invalide, email déjà existant, échec de l'authentification OAuth) et les mécanismes de reprise associés.

*[Insérer le diagramme d'activité de l'authentification]*

---

## VI- Modélisation des états

Les diagrammes précédents (séquence et activité) montrent le comportement du système du point de vue des **interactions** et des **processus**. Cette section s'intéresse à un angle complémentaire : le comportement d'un **objet individuel** au cours de sa vie dans le système.

Les diagrammes états-transitions UML modélisent les différents **états** qu'un objet peut traverser, les **événements** qui déclenchent les transitions entre ces états, et les **actions** exécutées lors de chaque transition. Ils sont particulièrement pertinents pour les objets dont le statut évolue de manière significative et conditionne le comportement du système.

Dans notre plateforme, deux entités possèdent un cycle de vie riche justifiant cette modélisation :

### 1) Diagramme états-transitions de PullRequest

Une Pull Request traverse les états suivants au cours de son cycle de vie :

- **Open** : état initial, la PR vient d'être synchronisée depuis le dépôt externe. Elle est visible dans le tableau de bord et peut être revendiquée par un développeur.
- **In_review** : une ou plusieurs revues ont été assignées. La PR est en cours d'examen par les reviewers désignés.
- **Approved** : état final positif, le reviewer a approuvé la PR. Le dépôt externe et le propriétaire sont notifiés, et des points sont attribués au reviewer et au développeur.
- **Rejected** : état final négatif, le reviewer a rejeté la PR. Le dépôt externe et le propriétaire sont notifiés, et des points sont attribués au reviewer uniquement.

Les transitions sont déclenchées par les méthodes des contrôleurs :
- `synchronizePRs()` → crée la PR à l'état **Open**
- `assignReviewers()` → passage de **Open** à **In_review**
- `finalizePullRequest(decision="approuver")` → passage à **Approved**
- `finalizePullRequest(decision="rejeter")` → passage à **Rejected**

*[Insérer le diagramme états-transitions de PullRequest]*

### 2) Diagramme états-transitions de Review

Une Review suit un cycle de vie en quatre états, définis par l'énumération `ReviewStatus` :

- **pendingReview** : état initial, la revue vient d'être créée suite à l'assignation d'un reviewer. Le reviewer est notifié par email et la revue apparaît dans sa liste de revues demandées.
- **acceptedForReview** : le reviewer a accepté de prendre en charge la revue. Il peut désormais consulter le code et rédiger son commentaire.
- **rejectedForReview** : le reviewer a décliné la demande de revue. Le développeur est notifié et peut choisir un autre reviewer.
- **completedReview** : état final, le reviewer a soumis son commentaire et sa décision. Des points lui sont attribués via le système de gamification.

Les transitions sont déclenchées par :
- `create(new Review(...))` + `notifyReviewRequest()` → création à l'état **pendingReview**
- `acceptReviewRequest()` → passage à **acceptedForReview**
- refus du reviewer → passage à **rejectedForReview**
- `submitReview()` → passage à **completedReview** avec attribution de points et mise à jour des badges

*[Insérer le diagramme états-transitions de Review]*

---

## VII- Architecture technique

Cette dernière section décrit l'architecture logicielle et physique du système. Après avoir modélisé **quoi** fait le système (cas d'utilisation), **comment** les objets interagissent (séquence), **comment** les processus s'enchaînent (activité) et **comment** les objets évoluent (états-transitions), nous décrivons ici **avec quoi** et **où** le système s'exécute.

### 1) Diagramme de composants

Le diagramme de composants représente l'organisation logicielle du système en modules cohérents et leurs dépendances. Il traduit le diagramme de packages en une vue plus concrète, orientée déploiement.

Notre système s'organise en quatre couches internes et trois systèmes externes :

**Couches internes :**
- **Interface (GUI)** : regroupe les 11 classes d'interface (LoginGUI, SignUpGUI, DashboardGUI, etc.) qui constituent le point de contact avec l'utilisateur.
- **Controller** : contient les 9 contrôleurs qui orchestrent la logique applicative. Ce composant dépend à la fois de la couche Interface (qui lui délègue les actions utilisateur) et de la couche Repository (à laquelle il délègue la persistance).
- **Repository** : regroupe les 7 repositories qui assurent l'accès aux données. Chaque repository est spécialisé pour une entité métier.
- **Entity** : contient les 6 classes métier (Account, DeveloperAccount, ReviewerAccount, AdminAccount, PullRequest, Review, AlgorithmParameters) qui représentent le domaine.

**Systèmes externes :**
- **Dépôts externes** (GitHub, GitLab, Bitbucket) : sources des Pull Requests, communiquant avec le système via le `SynchronisationController`.
- **Service Email** : utilisé par le `NotificationController` pour envoyer les notifications aux développeurs et reviewers.
- **Base de données** : stockage persistant des entités, accessible uniquement via les repositories.

Les dépendances respectent le principe de séparation en couches : l'interface ne communique qu'avec les contrôleurs, les contrôleurs ne communiquent qu'avec les repositories, et les repositories ne communiquent qu'avec les entités et la base de données.

*[Insérer le diagramme de composants]*

### 2) Diagramme de déploiement

Le diagramme de déploiement décrit la topologie physique du système : les nœuds matériels (machines, serveurs), les composants logiciels déployés sur chaque nœud, et les protocoles de communication entre eux.

Notre architecture se décompose en cinq nœuds :

- **Client** : la machine de l'utilisateur (navigateur web ou application desktop) hébergeant la couche GUI. Il communique avec le serveur d'application via **HTTPS**.

- **Serveur d'application** : le serveur central hébergeant les couches Controller, Repository et Entity. Il constitue le cœur du système et assure la coordination entre toutes les parties. Il communique avec :
  - la base de données (protocole interne),
  - les dépôts externes via **HTTPS** (APIs REST de GitHub, GitLab, Bitbucket),
  - le fournisseur email via **SMTP**.

- **Base de données** : le serveur de base de données hébergeant le stockage persistant des entités. Il est accessible uniquement depuis le serveur d'application, jamais directement depuis le client.

- **Dépôts externes** : les serveurs tiers (GitHub, GitLab, Bitbucket) qui hébergent les Pull Requests sources. La communication est bidirectionnelle : le système récupère les PRs (synchronisation) et notifie les décisions (finalisation).

- **Email Provider** : le serveur de messagerie (MailServer) utilisé pour l'envoi des notifications. La communication est unidirectionnelle : le système envoie des emails mais ne reçoit pas de réponses via ce canal.

Cette architecture respecte le modèle **client-serveur classique** avec une séparation nette entre la présentation (client), la logique métier (serveur d'application) et les données (base de données).

*[Insérer le diagramme de déploiement]*
