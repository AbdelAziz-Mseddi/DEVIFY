# Méthodes des classes Repository

## Introduction

Ce document définit les méthodes de chaque classe **Repository** en se basant sur les besoins réels des **contrôleurs**. Pour chaque méthode du repository, on indique quel contrôleur l'appelle et dans quel contexte, ce qui justifie son existence (principe YAGNI — *You Aren't Gonna Need It* : on ne crée pas de méthode "au cas où").

### Rappel du rôle des repositories

Les repositories sont la **couche d'accès aux données**. Ils :
- traduisent les opérations métier en requêtes sur la base de données,
- retournent des objets *entités* (jamais des objets bruts de la BD),
- ne contiennent **aucune logique métier** (pas de calcul de score, pas d'envoi d'email, pas de règles de gamification).

### Convention de nommage

- `findBy...()` → lecture (retourne une ou plusieurs entités)
- `save()` / `create()` → création
- `update...()` → mise à jour ciblée (un ou plusieurs champs)
- `delete...()` → suppression
- `exists...()` → vérification d'existence (retourne un booléen)
- `count...()` → comptage (retourne un entier)

---

## Classe abstraite `Repository`

Méthodes génériques héritées par tous les repositories concrets.

```
<<Abstract>>
Repository<T>
+ findAll() : List<T>
+ findById(id: int) : T
+ create(entity: T) : T
+ update(entity: T) : T
+ delete(id: int) : boolean
+ exists(id: int) : boolean
```

**Pourquoi ces méthodes de base ?** Presque tous les contrôleurs ont besoin de lire par ID, lister, créer et supprimer. Les mettre dans la classe abstraite évite la duplication.

---

## 1. `DeveloperAccountRepository`

### Méthodes spécifiques

```
+ findByEmail(email: String) : DeveloperAccount
+ findByNickName(nickName: String) : DeveloperAccount
+ existsByEmail(email: String) : boolean
+ updatePoints(uid: int, newPoints: int) : void
+ updateBadges(uid: int, badges: String[]) : void
+ incrementNbrPrs(uid: int) : void
+ addExternalAccountLink(uid: int, link: String) : void
+ removeExternalAccountLink(uid: int, link: String) : void
+ findByExternalAccountLink(link: String) : DeveloperAccount
+ updateRoleById(uid: int, newRole: String) : void
```

### Justification par contrôleur

| Méthode | Appelée par | Pour quoi faire |
|---|---|---|
| `findByEmail()` | `AuthentificationControleur.login()` | Identifier l'utilisateur lors de la connexion |
| `existsByEmail()` | `AuthentificationControleur.signUp()` | Vérifier qu'un email n'est pas déjà pris |
| `create()` *(hérité)* | `AuthentificationControleur.signUp()` | Créer le compte |
| `addExternalAccountLink()` | `AuthentificationControleur.linkExternalAccount()` | Ajouter une connexion GitHub/GitLab/Bitbucket |
| `findByExternalAccountLink()` | `SynchronisationControleur.synchronizePRs()` | Retrouver le dev à partir de son compte externe |
| `updatePoints()` | `GamificationControleur.awardPoints()` | Mettre à jour les points après une revue |
| `updateBadges()` | `GamificationControleur.grantBadge()` | Ajouter un nouveau badge |
| `incrementNbrPrs()` | `PullRequestControleur.claimPullRequest()` | Incrémenter le compteur de PRs ouvertes |
| `updateRoleById()` | `AdministrationControleur.changeRole()` | Transformer un dev en reviewer ou inverse |
| `delete()` *(hérité)* | `AdministrationControleur.deleteUser()` | Suppression d'utilisateur |

---

## 2. `ReviewerAccountRepository`

### Méthodes spécifiques

```
+ findByEmail(email: String) : ReviewerAccount
+ findAllOrderedByWorkload() : List<ReviewerAccount>
+ findAvailableReviewers() : List<ReviewerAccount>
+ updateNbrReviews(uid: int, newCount: int) : void
+ incrementNbrReviews(uid: int) : void
+ updateRating(uid: int, newRating: float) : void
+ findTopReviewers(limit: int) : List<ReviewerAccount>
+ countActiveReviewsByReviewer(uid: int) : int
```

### Justification par contrôleur

| Méthode | Appelée par | Pour quoi faire |
|---|---|---|
| `findByEmail()` | `AuthentificationControleur.login()` | Connexion du reviewer |
| `findAvailableReviewers()` | `AttributionControleur.suggestReviewers()` | Obtenir la liste sur laquelle appliquer l'algorithme |
| `countActiveReviewsByReviewer()` | `AttributionControleur.calculateScore()` | Calculer la charge de travail actuelle |
| `updateRating()` | `GamificationControleur.updateRating()` | Mise à jour de la note (après évaluation par un dev) |
| `incrementNbrReviews()` | `RevueControleur.finalizePullRequest()` | Incrémenter le nombre de revues faites |
| `findTopReviewers()` | `TableauDeBordControleur.loadDashboard()` | Afficher le classement éventuel |

> **Note** : `/rating` est un attribut dérivé dans ton diagramme. Il peut être stocké (cache) ou calculé à la volée. Si stocké → `updateRating()` est nécessaire.

---

## 3. `AdminRepository`

### Méthodes spécifiques

```
+ findByEmail(email: String) : AdminAccount
+ findAllAdmins() : List<AdminAccount>
+ countAdmins() : int
```

### Justification par contrôleur

| Méthode | Appelée par | Pour quoi faire |
|---|---|---|
| `findByEmail()` | `AuthentificationControleur.login()` | Connexion admin |
| `countAdmins()` | `AdministrationControleur.deleteUser()` | Empêcher la suppression du dernier admin |
| `create()` *(hérité)* | `AdministrationControleur` (création d'admin) | Ajouter un nouvel administrateur |

> **Note** : les admins ont peu de méthodes spécifiques car ils agissent sur les autres entités, pas sur eux-mêmes.

---

## 4. `PullRequestRepository`

### Méthodes spécifiques

```
+ findByDeveloper(uid: int) : List<PullRequest>
+ findByRepoName(repoName: String) : List<PullRequest>
+ findByStatus(status: String) : List<PullRequest>
+ findOpenPullRequests() : List<PullRequest>
+ findByPlatform(platformName: String) : List<PullRequest>
+ updateStatus(prid: int, newStatus: String) : void
+ updateClosedAt(prid: int, closedAt: Date) : void
+ saveOrUpdate(pr: PullRequest) : PullRequest
+ findByExternalId(externalId: String, platform: String) : PullRequest
+ deleteByRepoName(repoName: String) : void
```

### Justification par contrôleur

| Méthode | Appelée par | Pour quoi faire |
|---|---|---|
| `findByDeveloper()` | `PullRequestControleur.loadPullRequests()` | Lister les PRs d'un dev |
| `findOpenPullRequests()` | `TableauDeBordControleur.loadDashboard()` | Afficher les PRs ouvertes |
| `saveOrUpdate()` | `SynchronisationControleur.synchronizePRs()` | Créer si nouveau, mettre à jour si existant (*upsert*) |
| `findByExternalId()` | `SynchronisationControleur.synchronizePRs()` | Éviter les doublons lors de la synchro |
| `updateStatus()` | `RevueControleur.finalizePullRequest()` | Passer à "approved" ou "rejected" |
| `updateClosedAt()` | `RevueControleur.finalizePullRequest()` | Enregistrer la date de fermeture |
| `findByRepoName()` | `PullRequestControleur.importRepository()` | Charger les PRs d'un dépôt spécifique |
| `findByPlatform()` | `SynchronisationControleur` | Synchroniser par plateforme (GitHub, GitLab...) |

> **Point clé** : `saveOrUpdate()` est crucial pour la synchronisation — on ne sait pas si la PR est nouvelle ou existante tant qu'on n'a pas vérifié.

---

## 5. `ReviewRepository`

### Méthodes spécifiques

```
+ findByReviewer(uid: int) : List<Review>
+ findByPullRequest(prid: int) : List<Review>
+ findPendingReviewsByReviewer(uid: int) : List<Review>
+ findByStatus(status: String) : List<Review>
+ findByReviewerAndStatus(uid: int, status: String) : List<Review>
+ updateStatus(rvid: int, newStatus: String) : void
+ updateComment(rvid: int, comment: String) : void
+ countCompletedReviewsByReviewer(uid: int) : int
+ findReviewHistory(prid: int) : List<Review>
+ findRecentReviewsByReviewer(uid: int, limit: int) : List<Review>
```

### Justification par contrôleur

| Méthode | Appelée par | Pour quoi faire |
|---|---|---|
| `findPendingReviewsByReviewer()` | `RevueControleur.loadRequestedReviews()` | Afficher les revues demandées (en attente) |
| `findReviewHistory()` | `RevueControleur.getReviewDetails()` | Afficher l'historique de discussion d'une PR |
| `create()` *(hérité)* | `PullRequestControleur.assignReviewers()` | Créer une demande de revue |
| `updateStatus()` | `RevueControleur.submitReview()` | Passer à "approved" / "rejected" / "postponed" |
| `updateComment()` | `RevueControleur.submitReview()` | Sauvegarder le commentaire |
| `countCompletedReviewsByReviewer()` | `AttributionControleur.calculateScore()` | Paramètre d'historique dans l'algorithme |
| `findRecentReviewsByReviewer()` | `TableauDeBordControleur.loadDashboard()` | Afficher l'activité récente |
| `findByReviewerAndStatus()` | `RevueControleur.loadRequestedReviews()` | Filtrer (pending, done, etc.) |

---

## 6. `AssignAlgorithmRepository`

### Méthodes spécifiques

```
+ findActiveAlgorithm() : AssignAlgorithm
+ updateWeights(currentWorkLoad: float, reviewerRating: float, points: float, badges: float) : void
+ findHistoricalConfigurations() : List<AssignAlgorithm>
+ saveConfigurationSnapshot(algo: AssignAlgorithm) : void
```

### Justification par contrôleur

| Méthode | Appelée par | Pour quoi faire |
|---|---|---|
| `findActiveAlgorithm()` | `AttributionControleur.getCurrentAlgorithm()` | Charger les pondérations actives pour le calcul de score |
| `updateWeights()` | `AdministrationControleur.updateAlgorithmParameters()` | L'admin modifie les pondérations |
| `saveConfigurationSnapshot()` | `AdministrationControleur.updateAlgorithmParameters()` | Historiser avant chaque changement (audit) |
| `findHistoricalConfigurations()` | `AdministrationControleur` | Voir l'évolution des paramètres |

> **Note** : il n'y a **qu'une seule configuration active** à un instant donné. `findActiveAlgorithm()` retourne un singleton métier.

---

## 📊 Synthèse : matrice contrôleur → repository

Cette matrice montre quels contrôleurs parlent à quels repositories. Elle aide à visualiser les dépendances.

| | `DevAccRepo` | `RevAccRepo` | `AdminRepo` | `PRRepo` | `ReviewRepo` | `AlgoRepo` |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **AuthentificationControleur** | ✅ | ✅ | ✅ | | | |
| **SynchronisationControleur** | ✅ | | | ✅ | | |
| **PullRequestControleur** | ✅ | | | ✅ | ✅ | |
| **AttributionControleur** | | ✅ | | | ✅ | ✅ |
| **RevueControleur** | | ✅ | | ✅ | ✅ | |
| **GamificationControleur** | ✅ | ✅ | | | | |
| **NotificationControleur** | ✅ | ✅ | | | | |
| **TableauDeBordControleur** | ✅ | ✅ | | ✅ | ✅ | |
| **AdministrationControleur** | ✅ | ✅ | ✅ | | | ✅ |

---

## 🎯 Principes respectés

### 1. YAGNI (*You Aren't Gonna Need It*)
Chaque méthode a au moins un contrôleur qui l'utilise. Aucune méthode "au cas où".

### 2. Responsabilité unique
Les repositories **ne font que** de la persistance. Le calcul de score est dans `AttributionControleur`, pas dans `ReviewerAccountRepository`.

### 3. Encapsulation des requêtes
Les contrôleurs ne connaissent pas le SQL. Ils appellent `findPendingReviewsByReviewer(uid)` sans savoir si c'est une jointure, un index, ou une vue.

### 4. Opérations ciblées vs complètes
Quand on change un seul champ, on utilise `updateX()` plutôt que `update()` complet. Exemple :

✅ `updatePoints(uid, 15)` → ne met à jour que la colonne `points`
❌ Charger le reviewer entier, modifier, puis `update(reviewer)` → coûteux et sujet aux conflits

### 5. Noms explicites
`findPendingReviewsByReviewer()` est plus clair que `findByReviewerWithStatus("pending")`.

---

## 💡 Remarques d'implémentation

### Transactions
Certaines opérations doivent être atomiques. Par exemple, dans `RevueControleur.finalizePullRequest()` :
- `reviewRepo.updateStatus()`
- `pullRequestRepo.updateStatus()`
- `reviewerRepo.incrementNbrReviews()`
- `developerRepo.updatePoints()` (si applicable)

Ces appels doivent être **dans une même transaction** pour garantir la cohérence. La gestion transactionnelle est typiquement faite au niveau du contrôleur (via `@Transactional` en Spring, par exemple).

### Pagination
Pour `findAll()` sur des grosses tables (PRs, Reviews), prévoir une version paginée :
```
+ findAll(page: int, size: int) : Page<T>
```

### Requêtes dérivées dynamiques
Si le framework le permet (Spring Data JPA, par exemple), certaines méthodes peuvent être générées automatiquement à partir de leur nom, évitant d'écrire le SQL.

---

*Document généré dans le cadre du projet CSI - Plateforme de validation collaborative de pull requests (GL2/3)*
