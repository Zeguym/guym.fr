---
title: AsSplitQuery — maîtriser le chargement de plusieurs relations (EFcore)
tags: dotnet efcore performance split-query
series: "Entity Framework"
---

Entity Framework Core charge par défaut toutes les relations avec une seule requête SQL. Cette approche peut provoquer une **explosion cartésienne** lorsque plusieurs collections sont incluses. `AsSplitQuery` résout ce problème en découpant la requête en plusieurs instructions SQL distinctes.

<!--more-->
 {% include series.html %}

# 1. Le problème : l'explosion cartésienne

Lorsqu'on inclut plusieurs collections avec `.Include()`, EF Core génère par défaut une unique requête SQL qui effectue des jointures entre toutes les tables.

## Exemple

```csharp
var commandes = db.CdeClients
    .Include(c => c.Lignes)
    .Include(c => c.Paiements)
    .ToList();
```

EF Core génère une requête de ce type :

```sql
SELECT c.*, l.*, p.*
FROM CdeClients c
LEFT JOIN LigCdeClients l ON l.CdeClientId = c.Id
LEFT JOIN Paiements p     ON p.CdeClientId = c.Id
```

## Le piège du produit cartésien

Supposons qu'une commande ait **3 lignes** et **2 paiements**. La jointure produit **3 × 2 = 6 lignes** dans le jeu de résultats, même si les données n'en contiennent que 5.

| CdeClientId | LigneId | PaiementId |
|-------------|---------|------------|
| 1           | L1      | P1         |
| 1           | L1      | P2         |
| 1           | L2      | P1         |
| 1           | L2      | P2         |
| 1           | L3      | P1         |
| 1           | L3      | P2         |

EF Core déduplique correctement ces données, mais le volume **transféré sur le réseau** est multiplié. Avec de grandes collections, cela peut sérieusement dégrader les performances.

> Le terme **explosion cartésienne** vient du produit cartésien mathématique : le nombre de lignes résultantes est le **produit** des cardinalités de chaque collection incluse.

---

# 2. La solution : `AsSplitQuery()`

`AsSplitQuery()` indique à EF Core de découper la requête en **plusieurs requêtes SQL distinctes**, une par collection incluse. EF Core recompose ensuite les données en mémoire.

```csharp
var commandes = db.CdeClients
    .Include(c => c.Lignes)
    .Include(c => c.Paiements)
    .AsSplitQuery()
    .ToList();
```

EF Core génère alors trois requêtes séparées :

```sql
-- Requête 1 : entités principales
SELECT c.*
FROM CdeClients c;

-- Requête 2 : première collection
SELECT l.*
FROM LigCdeClients l
WHERE l.CdeClientId IN (1, 2, 3, ...);

-- Requête 3 : deuxième collection
SELECT p.*
FROM Paiements p
WHERE p.CdeClientId IN (1, 2, 3, ...);
```

Chaque requête retourne exactement les lignes nécessaires, sans duplication.

---

# 3. `AsSingleQuery()` — revenir au comportement par défaut

À l'inverse, `AsSingleQuery()` force EF Core à utiliser une seule requête, même si le comportement global a été changé (voir section 5).

```csharp
var commandes = db.CdeClients
    .Include(c => c.Lignes)
    .AsSingleQuery()
    .ToList();
```

---

# 4. Quand utiliser `AsSplitQuery` ?

| Situation | Recommandation |
|-----------|----------------|
| Inclusion de **plusieurs collections** volumineuses | `AsSplitQuery` |
| Inclusion d'**une seule collection** ou de peu de données | `AsSingleQuery` (défaut) |
| **Cohérence transactionnelle** requise entre les requêtes | `AsSingleQuery` |
| Requête avec **filtres, tri ou pagination** sur les collections incluses | Évaluer les deux et mesurer |

> **Règle pratique** : dès qu'une requête inclut au moins deux collections et que les performances sont un enjeu, envisager `AsSplitQuery`.

---

# 5. Configuration globale

Il est possible de configurer `AsSplitQuery` comme comportement par défaut pour l'ensemble du `DbContext`, sans avoir à l'ajouter requête par requête.

```csharp
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
        sqlOptions.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)));
```

Dans ce cas, toutes les requêtes utiliseront automatiquement les requêtes divisées, sauf celles explicitement marquées `.AsSingleQuery()`.

---

# 6. Points d'attention et limitations

## 6.1. Cohérence des données

Avec `AsSplitQuery`, les plusieurs requêtes SQL sont exécutées dans des **appels distincts** à la base de données. Si des données sont modifiées entre deux requêtes (par un autre processus), les résultats peuvent être **incohérents**.

Pour garantir la cohérence, il faut envelopper les requêtes dans une transaction explicite :

```csharp
using var transaction = await db.Database.BeginTransactionAsync();

var commandes = await db.CdeClients
    .Include(c => c.Lignes)
    .Include(c => c.Paiements)
    .AsSplitQuery()
    .ToListAsync();

await transaction.CommitAsync();
```

## 6.2. Plusieurs allers-retours réseau

`AsSplitQuery` génère **N + 1 requêtes** (1 pour l'entité principale, 1 par collection incluse). Cela augmente le nombre d'allers-retours vers la base de données. Sur un réseau à forte latence, cela peut annuler le gain obtenu.

## 6.3. Incompatibilité avec certaines opérations

`AsSplitQuery` n'est pas compatible avec toutes les situations. Par exemple, les requêtes utilisant des opérateurs tels que `Distinct`, `GroupBy`, `Skip`/`Take` sur les entités principales peuvent produire des comportements inattendus ou des erreurs.

```csharp
// Attention : combinaison potentiellement problématique
var commandes = db.CdeClients
    .Include(c => c.Lignes)
    .OrderBy(c => c.Date)
    .Skip(10).Take(5)   // pagination sur l'entité principale
    .AsSplitQuery()     // peut générer un avertissement ou une erreur
    .ToList();
```

> EF Core émet un avertissement dans ce cas. Il est conseillé de tester et de vérifier le SQL généré avec `.ToQueryString()`.

## 6.4. Inspecter le SQL généré

Pour vérifier les requêtes produites avant exécution :

```csharp
var query = db.CdeClients
    .Include(c => c.Lignes)
    .Include(c => c.Paiements)
    .AsSplitQuery();

Console.WriteLine(query.ToQueryString());
```

---

# 7. Résumé

| | `AsSingleQuery` | `AsSplitQuery` |
|---|---|---|
| **Requêtes SQL générées** | 1 | N (1 + nb de collections) |
| **Risque de produit cartésien** | Oui | Non |
| **Cohérence transactionnelle** | Garantie | À gérer manuellement |
| **Allers-retours réseau** | 1 | N |
| **Recommandé si** | Peu de collections, faible volume | Plusieurs collections volumineuses |

`AsSplitQuery` est un outil précieux pour éviter les dégradations de performance liées au produit cartésien. Il convient toutefois de peser les compromis, notamment en matière de cohérence et de latence réseau, avant de l'adopter globalement.
