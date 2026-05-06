# Architecture MVC : Comprendre les différents types de classes

## Introduction

Dans la conception orientée objet, on distingue plusieurs types de classes selon leur **responsabilité**. Cette séparation (issue de MVC et de la méthode *Robustness Analysis*) permet d'obtenir un code maintenable, testable et évolutif.

Les quatre types principaux sont :
- **Entités** (Entity / Modèle)
- **Interfaces** (Boundary / GUI / Vue)
- **Contrôleurs** (Control)
- **Repositories** (DAO / Couche de persistance)

---

## 🍽️ L'analogie du restaurant

Pour bien comprendre, imaginons un restaurant :

| Élément du restaurant | Type de classe | Rôle |
|---|---|---|
| Le **client** | Utilisateur | Celui qui passe commande |
| La **salle** (tables, menu, déco) | **Boundary / GUI** | Ce que le client voit et touche |
| Le **serveur** | **Contrôleur** | Orchestre, transmet, coordonne |
| Le **plat, la recette, les ingrédients** | **Entité** | Les données et concepts métier |
| La **cuisine, le frigo, le stock** | **Repository** | Là où les données sont stockées |

### La règle d'or

> **Le client ne va jamais directement en cuisine.**
> Il parle au serveur, qui parle à la cuisine.

Si le client entrait lui-même en cuisine pour prendre son plat, ce serait le chaos : pas de contrôle, pas de cohérence, pas de sécurité. C'est exactement ce qui arrive quand une interface parle directement à la base de données — le code devient ingérable.

---

## 1. Les classes Entité (Entity / Modèle)

### Rôle
Représenter les **concepts métier** et leurs données. Ce sont les "noms communs" du domaine.

### Contenu
- Des **attributs** (les données)
- Des méthodes qui concernent **uniquement** l'objet lui-même

### Ce qu'elles ne font PAS
- Elles ne parlent pas à l'utilisateur
- Elles ne parlent pas à la base de données
- Elles ne parlent pas aux systèmes externes

### Test simple
> *"Est-ce que ce concept existerait même si l'application n'existait pas ?"*
>
> Si **oui** → c'est une entité.

### Exemples concrets
`PullRequest`, `Review`, `DeveloperAccount`, `ReviewerAccount`, `AssignAlgorithm`.

Une Pull Request existe dans GitHub sans notre application — c'est un concept métier pur.

### Bon vs mauvais usage

✅ **Méthode légitime** :
```
Review.calculateDuration()
```
Retourne la durée entre `createdAt` et maintenant. Elle concerne la review elle-même.

❌ **Méthode illégitime** :
```
Review.saveToDatabase()
```
Ce n'est pas le rôle de la review de savoir comment elle est stockée.

---

## 2. Les classes Interface (Boundary / GUI / Vue)

### Rôle
Gérer tout ce qui est **affiché** à l'utilisateur et toutes les **interactions** avec lui (clics, formulaires, navigation).

### Contenu
- Des composants graphiques (boutons, listes, formulaires)
- Des méthodes qui réagissent aux événements utilisateur

### Ce qu'elles ne font PAS
- Elles ne prennent **aucune décision métier**
- Quand on clique sur "Approuver", la GUI ne sait pas ce que ça veut dire "approuver" — elle transmet juste l'information au contrôleur

### Test simple
> *"Si je change de technologie d'affichage (web → mobile → CLI), est-ce que cette classe doit être réécrite ?"*
>
> Si **oui** → c'est une boundary.

### Exemples concrets
`LoginGUI`, `DashboardGUI`, `PullRequestGUI`, `ReviewGUI`.

Si demain on passe de React à une app mobile, on jette toutes ces classes — mais pas les entités ni les contrôleurs.

### Bon vs mauvais usage

✅ **Exemple légitime** :
```
ReviewGUI.submitReview()
→ collecte les données du formulaire
→ appelle RevueControleur.submitReview(...)
```

❌ **Exemple illégitime** :
```
ReviewGUI qui calcule directement les points du reviewer
```
C'est de la logique métier, pas de l'affichage.

---

## 3. Les classes Contrôle (Controller)

### Rôle
**Orchestrer** les cas d'utilisation. Elles font le pont entre ce que l'utilisateur veut faire (boundary) et ce que ça implique sur les données (entity + repository).

### Contenu
- De la **logique applicative** : enchaînement d'étapes, coordination entre plusieurs entités
- Des **règles métier transversales** qui ne rentrent dans aucune entité spécifique
- Des **appels aux repositories** pour sauvegarder/charger

### Ce qu'elles ne font PAS
- Elles n'affichent rien
- Elles ne stockent pas directement les données (elles délèguent aux repositories)

### Test simple
> *"Ce traitement implique-t-il plusieurs objets ou un enchaînement d'étapes ?"*
>
> Si **oui** → contrôleur.

### Exemple phare
```
AttributionControleur.suggestReviewers(pr)
```
Il faut :
1. Récupérer tous les reviewers
2. Calculer un score pour chacun (combinant plusieurs paramètres)
3. Trier les résultats
4. Retourner les 5 meilleurs

Ce n'est le boulot ni de la PR, ni du Reviewer seul, ni d'un écran.

### Analogie
Le serveur qui, pour une commande complexe, prévient la cuisine chaude, la cuisine froide et le bar, puis synchronise tout pour servir en même temps.

---

## 4. Les classes Repository (DAO / Persistance)

### Rôle
Gérer la **persistance** : sauvegarder et récupérer les entités dans une base de données (ou un fichier, ou une API).

### Contenu
- `findById()`, `findAll()`, `save()`, `delete()`, `findByXxx()`
- Les requêtes SQL ou appels à l'ORM

### Ce qu'elles ne font PAS
- Aucune logique métier
- Un `PullRequestRepository` ne sait pas ce qu'est une "bonne" PR — il sait juste les stocker et les retrouver

### Pourquoi les séparer des entités ?
Parce que si on change de base de données (MySQL → MongoDB), on ne veut réécrire que les repositories, pas toutes les entités.

### Exemples concrets
`PullRequestRepository`, `ReviewRepository`, `DeveloperAccountRepository`.

---

## 🔄 Le flux complet : exemple d'un scénario

**Scénario : Un reviewer approuve une PR**

```
1. [GUI]        ReviewerReviewGUI
                └─ Le reviewer clique sur "Approuver", tape un commentaire
                └─ Appelle RevueControleur.finalizePullRequest(review, "approved", comment)

2. [CONTROL]    RevueControleur
                ├─ Met à jour review.status = "approved"
                ├─ Appelle ReviewRepository.save(review)         ──> [REPOSITORY]
                ├─ Appelle SynchronisationControleur (notifier GitHub)
                ├─ Appelle NotificationControleur (email propriétaire)
                └─ Appelle GamificationControleur.awardPoints(reviewer)

3. [ENTITY]     Review, PullRequest, ReviewerAccount
                └─ Leurs attributs sont mis à jour en mémoire

4. [REPOSITORY] ReviewRepository, ReviewerAccountRepository
                └─ Écrivent les changements en base de données

5. [GUI]        DashboardGUI
                └─ Rafraîchit pour montrer la nouvelle liste
```

### Observation clé

**Chaque classe reste dans son rôle** :
- La GUI n'a jamais touché à la base
- L'entité n'a jamais envoyé d'email
- Le repository n'a jamais décidé d'attribuer des points

---

## 📊 Tableau récapitulatif

| Couche | Question qu'elle répond | Exemple | Ne doit **jamais** faire |
|---|---|---|---|
| **Entity** | *"Qu'est-ce que c'est ?"* | `PullRequest`, `Review` | Afficher, sauvegarder, appeler APIs externes |
| **Boundary / GUI** | *"Comment l'utilisateur voit et agit ?"* | `DashboardGUI` | Calculer des règles métier, toucher à la base |
| **Control** | *"Comment orchestrer un cas d'utilisation ?"* | `AttributionControleur` | Afficher, stocker directement |
| **Repository** | *"Comment on lit/écrit les données ?"* | `ReviewRepository` | Logique métier, interface utilisateur |

---

## 💡 Principe clé : "Tell, Don't Ask"

Quand une classe doit faire quelque chose, on lui **dit** de le faire plutôt que de lui demander ses données pour agir à sa place.

❌ **Mauvais** (logique dans le contrôleur, entité anémique) :
```
int points = reviewer.getPoints();
points += 10;
reviewer.setPoints(points);
```

✅ **Bon** (logique dans l'entité) :
```
reviewer.addPoints(10);
```

Ce principe permet de garder chaque classe **responsable de sa propre cohérence**.

---

## 🎯 Résumé en une phrase par couche

- **Entité** : *"Je représente un concept métier et ses données."*
- **Boundary / GUI** : *"Je montre et je capture les interactions utilisateur."*
- **Contrôle** : *"J'orchestre les étapes d'un cas d'utilisation."*
- **Repository** : *"Je lis et j'écris les données dans le stockage."*

---

## 🏗️ Avantages de cette séparation

1. **Maintenabilité** : une modification dans une couche n'impacte pas les autres
2. **Testabilité** : on peut tester chaque couche indépendamment
3. **Évolutivité** : on peut changer de technologie (framework, BD, UI) sans tout réécrire
4. **Clarté** : chaque développeur sait où chercher et où ajouter du code
5. **Réutilisabilité** : les entités et contrôleurs peuvent être réutilisés avec différentes UI

---

*Document généré dans le cadre du projet CSI - Plateforme de validation collaborative de pull requests (GL2/3)*
