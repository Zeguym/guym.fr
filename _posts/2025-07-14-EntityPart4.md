---
title: Bonnes pratiques pour les requêtes (EF Core)
tags: dotnet efcore clean-queries 
series: "Entity Framework"
---

Entity Framework Core : bonnes pratiques pour écrire des requêtes performantes et éviter les pièges courants

<!--more-->
 {% include series.html %}
# Bonnes pratiques à appliquer

Une requête LINQ n'est qu'une description : c'est le SQL généré qui s'exécute réellement. Pour chaque pratique, on regarde donc le SQL produit par EF Core 10 sur SQL Server. Pour l'afficher soi-même, deux options :

```csharp
// 1. Sans exécuter la requête
var sql = db.Clients.Where(c => c.IsActive).ToQueryString();

// 2. En journalisant toutes les commandes envoyées à la base
optionsBuilder
    .UseSqlServer(connectionString)
    .LogTo(Console.WriteLine, LogLevel.Information);
```

## 1. Privilégier les requêtes côté serveur

- Utiliser des méthodes LINQ qui se traduisent en SQL (`Where`, `Select`, `OrderBy`, etc.).
- Éviter d’appeler des méthodes .NET non traduites en SQL avant d’avoir récupéré les données (par exemple : `.ToList()`, `.AsEnumerable()`).

```csharp
// ❌ Toute la table est chargée, le filtre s'applique en mémoire
var actifs = db.Clients.AsEnumerable().Where(c => c.IsActive).ToList();

// ✅ Le filtre est traduit en SQL
var actifs = db.Clients.Where(c => c.IsActive).ToList();
```

```sql
-- ❌ Version AsEnumerable : aucune clause WHERE
SELECT [c].[Id], [c].[Name], [c].[Email], [c].[IsActive]
FROM [Clients] AS [c]

-- ✅ Version IQueryable
SELECT [c].[Id], [c].[Name], [c].[Email], [c].[IsActive]
FROM [Clients] AS [c]
WHERE [c].[IsActive] = CAST(1 AS bit)
```

> Depuis EF Core 3, une méthode non traduisible dans un `Where` lève une `InvalidOperationException` (« could not be translated ») au lieu d'être évaluée silencieusement côté client. Seul le `Select` final peut contenir du code .NET non traduisible.

## 2. Appliquer le filtrage le plus tôt possible

- Placer les `.Where()` avant les `.Select()` ou `.OrderBy()` afin de limiter le volume de données transférées.

Sur un `IQueryable`, EF Core regroupe les opérateurs en une seule requête : l'ordre `Where`/`OrderBy` change peu le SQL. La règle est surtout cruciale **avant toute matérialisation** (`ToList`, `AsEnumerable`, `foreach`) : tout ce qui est après s'exécute en mémoire.

## 3. Projections avec `Select`

- Utiliser `.Select()` pour ne récupérer que les colonnes nécessaires (DTO, ViewModel).
- Exemple :

```csharp
var clients = db.Clients
    .Where(c => c.IsActive)
    .Select(c => new ClientDto { Id = c.Id, Name = c.Name })
    .ToList();
```

```sql
SELECT [c].[Id], [c].[Name]
FROM [Clients] AS [c]
WHERE [c].[IsActive] = CAST(1 AS bit)
```

Seules deux colonnes transitent sur le réseau, et les entités projetées ne sont pas suivies par le Change Tracker.

## 4. Pagination

- Utiliser `.Skip()` et `.Take()` pour effectuer la pagination côté SQL.

```csharp
var page = db.Products
    .OrderBy(p => p.Name)
    .Skip(pageIndex * pageSize)
    .Take(pageSize)
    .ToList();
```

```sql
SELECT [p].[Id], [p].[Name], [p].[Price]
FROM [Products] AS [p]
ORDER BY [p].[Name]
OFFSET @p ROWS FETCH NEXT @pageSize ROWS ONLY
```

Pièges :

- ❌ Paginer sans `OrderBy` : l'ordre des lignes n'est pas garanti, une même ligne peut apparaître sur deux pages (EF Core émet un avertissement).
- ❌ Trier sur une colonne non unique (`Name`) : ajouter une clé de départage, `.OrderBy(p => p.Name).ThenBy(p => p.Id)`.
- Sur de très grandes tables, `OFFSET` devient coûteux sur les pages lointaines : la pagination par clé (`Where(p => p.Id > lastId).Take(pageSize)`) est alors préférable.

## 5. Chargement des relations

- Privilégier l’utilisation de `.Include()` pour charger explicitement les relations nécessaires.

```csharp
var orders = db.Orders.Include(o => o.Customer).ToList();
```

```sql
SELECT [o].[Id], [o].[CustomerId], [o].[OrderDate], [c].[Id], [c].[Name]
FROM [Orders] AS [o]
INNER JOIN [Customers] AS [c] ON [o].[CustomerId] = [c].[Id]
```

La relation étant obligatoire (`CustomerId` non nullable), EF Core génère un `INNER JOIN` ; pour une relation optionnelle, ce serait un `LEFT JOIN`.

- Éviter le chargement paresseux (lazy loading) lorsque l’on souhaite garder un contrôle précis sur les requêtes exécutées.

## 6. Éviter les requêtes N+1

Le problème N+1 survient lorsqu’une requête est exécutée pour une entité principale, puis une requête supplémentaire pour chaque entité liée.

- Utiliser `.Include()` ou des projections pour éviter de charger les entités liées une par une.

### Exemple du problème N+1

Supposons que l’on souhaite récupérer une liste de commandes et, pour chaque commande, ses lignes :

```csharp
// Provoque N+1 requêtes SQL !
var commandes = db.CdeClients.ToList();
foreach (var commande in commandes)
{
    var lignes = db.LigCdeClients.Where(l => l.CdeClientId == commande.Id).ToList();
    // ...
}
```

- Ici, 1 requête pour les commandes, puis 1 requête par commande pour les lignes.

```sql
SELECT [c].[Id], [c].[Date] FROM [CdeClients] AS [c]

-- puis, répété N fois avec une valeur différente :
SELECT [l].[Id], [l].[CdeClientId], [l].[Quantite]
FROM [LigCdeClients] AS [l]
WHERE [l].[CdeClientId] = @Id
```

### Bonne pratique : charger les relations en une seule requête

Utiliser `.Include()` pour charger les entités liées en une seule requête SQL :

```csharp
var commandes = db.CdeClients
    .Include(c => c.Lignes) // navigation property
    .ToList();
```

- Résultat : 1 seule requête SQL qui ramène les commandes **et** leurs lignes.

```sql
SELECT [c].[Id], [c].[Date], [l].[Id], [l].[CdeClientId], [l].[Quantite]
FROM [CdeClients] AS [c]
LEFT JOIN [LigCdeClients] AS [l] ON [c].[Id] = [l].[CdeClientId]
ORDER BY [c].[Id]
```

> Attention : les colonnes de la commande sont répétées pour chaque ligne. Avec plusieurs `Include` de collections, le volume explose (« explosion cartésienne ») ; `AsSplitQuery()` permet alors de découper en plusieurs requêtes.

### Alternative avec projection

Pour un DTO, il est possible de projeter les données de la manière suivante :

```csharp
var commandes = db.CdeClients.Select(c => new CommandeDto
{
    Id = c.Id,
    Lignes = c.LigCdeClients.Select(l => new LigneDto { Id = l.Id, ... }).ToList()
})
.ToList();
```

Dans ce cas, Entity Framework générera une seule requête SQL avec jointure.

```sql
SELECT [c].[Id], [l].[Id], [l].[Quantite]
FROM [CdeClients] AS [c]
LEFT JOIN [LigCdeClients] AS [l] ON [c].[Id] = [l].[CdeClientId]
ORDER BY [c].[Id]
```

## 7. Utiliser `Any()` et `Count()` de manière appropriée

- Préférer `Any()` pour vérifier l’existence, généralement plus performant que `Count() > 0`.

```csharp
bool hasOrders = db.Orders.Any(o => o.CustomerId == id);
```

```sql
-- ✅ Any() : EXISTS s'arrête dès la première ligne trouvée
SELECT CASE
    WHEN EXISTS (
        SELECT 1
        FROM [Orders] AS [o]
        WHERE [o].[CustomerId] = @id) THEN CAST(1 AS bit)
    ELSE CAST(0 AS bit)
END

-- ❌ Count() > 0 : toutes les lignes correspondantes sont comptées
SELECT COUNT(*)
FROM [Orders] AS [o]
WHERE [o].[CustomerId] = @id
```

Il convient toutefois de rester attentif aux utilisations de `Any()` et `Count()` susceptibles de générer des sous-requêtes (et donc des requêtes de type N+1).

```csharp
 db.Customers.Where(c => c.Orders.Any(o => o.IsPaid));
```

```sql
SELECT [c].[Id], [c].[Name]
FROM [Customers] AS [c]
WHERE EXISTS (
    SELECT 1
    FROM [Orders] AS [o]
    WHERE [c].[Id] = [o].[CustomerId] AND [o].[IsPaid] = CAST(1 AS bit))
```

Ici, EF Core produit une seule requête avec une sous-requête **corrélée** : c'est la base qui l'évalue pour chaque client. Avec un index sur `Orders.CustomerId`, c'est généralement efficace ; sans index, le coût grimpe vite.

Dans certains cas, une requête explicite avec jointure, par exemple en sélectionnant le client et en filtrant sur les commandes payées, sera plus efficace qu’une sous-requête répétée.

Le vrai N+1 apparaît quand le `Any()`/`Count()` est appelé **en mémoire**, dans une boucle, sur une navigation chargée en lazy loading :

```csharp
// ❌ Une requête par client si Orders est en lazy loading
foreach (var c in db.Customers.ToList())
    if (c.Orders.Count > 0) { /* ... */ }
```

## 8. Utiliser `AsNoTracking()` pour la lecture

- Pour les requêtes en lecture seule, ajouter `.AsNoTracking()` permet d’améliorer les performances.

```csharp
var products = db.Products.AsNoTracking().ToList();
```

Le SQL est **identique** avec ou sans `AsNoTracking()` : le gain se situe côté .NET (pas de snapshot des valeurs, pas d'enregistrement dans le Change Tracker, moins d'allocations). Une projection `Select` vers un DTO n'est de toute façon pas suivie.

## 9. Éviter les requêtes complexes côté client

- Éviter les jointures ou les transformations complexes après l’appel à `.ToList()`. Il est préférable de conserver le maximum de logique côté SQL.

## 10. Utiliser des expressions lambda claires

- Privilégier des expressions lisibles et nommer les variables de manière explicite.

## 11. Utiliser les méthodes asynchrones

Il est recommandé d’utiliser les méthodes asynchrones (`async`) avec Entity Framework pour plusieurs raisons :

- **Performance et scalabilité** : les méthodes asynchrones (`ToListAsync`, `SaveChangesAsync`, etc.) libèrent le thread pendant l’attente des opérations d’E/S (accès à la base), ce qui permet au serveur de traiter d’autres requêtes.
- **Réactivité** : dans les applications web ou desktop, l’utilisation d’`async` évite de bloquer l’interface utilisateur ou les threads du serveur.
- **Bonne pratique moderne** : le modèle asynchrone est particulièrement adapté aux architectures cloud et microservices, où la gestion efficace des ressources est essentielle.

L’utilisation systématique des méthodes asynchrones avec Entity Framework contribue à améliorer les performances, la réactivité et la capacité à traiter un grand nombre de requêtes simultanément.

```csharp
var clients = await db.Clients
    .Where(c => c.IsActive)
    .ToListAsync(cancellationToken);
```

> Un `DbContext` n'est pas thread-safe : ne jamais lancer deux requêtes en parallèle (`Task.WhenAll`) sur la même instance. Toujours faire `await` avant la requête suivante.

# Pièges courants

## Comparer des chaînes avec `StringComparison`

```csharp
// ❌ Non traduisible : InvalidOperationException
db.Clients.Where(c => c.Name.Equals(name, StringComparison.OrdinalIgnoreCase));

// ✅ La sensibilité à la casse dépend de la collation de la colonne (insensible par défaut sur SQL Server)
db.Clients.Where(c => c.Name == name);
```

## `Contains` sur une liste

```csharp
int[] ids = [1, 2, 3];
var produits = await db.Products.Where(p => ids.Contains(p.Id)).ToListAsync();
```

En EF Core 10, chaque valeur devient un paramètre (la liste est « paddée » pour limiter le nombre de variantes de requêtes) :

```sql
SELECT [p].[Id], [p].[Name], [p].[Price]
FROM [Products] AS [p]
WHERE [p].[Id] IN (@ids1, @ids2, @ids3)
```

Les versions 8 et 9 envoyaient un seul paramètre JSON décodé par `OPENJSON` ; le mode peut être choisi avec `UseParameterizedCollectionMode(...)`. Pour des listes de plusieurs milliers d'éléments, mieux vaut passer par une table temporaire ou une jointure.

## Réutiliser un `IQueryable` en croyant qu'il est déjà exécuté

```csharp
var query = db.Orders.Where(o => o.IsPaid);
var total = await query.CountAsync();   // requête 1
var items = await query.ToListAsync();  // requête 2 : la requête est rejouée
```

Un `IQueryable` est une requête différée : chaque matérialisation repart en base.

# Résumé

| Pratique | SQL généré | Gain |
|---|---|---|
| Filtrer avant matérialisation | `WHERE` côté serveur | Moins de lignes transférées |
| Projection `Select` | `SELECT` des seules colonnes utiles | Moins de colonnes, pas de tracking |
| `Skip`/`Take` + `OrderBy` | `OFFSET ... FETCH NEXT` | Pagination déterministe en base |
| `Include` | `INNER`/`LEFT JOIN` | Évite le N+1 (attention aux collections multiples) |
| `Any()` plutôt que `Count() > 0` | `EXISTS` | Arrêt à la première ligne |
| `AsNoTracking()` | Identique | Moins de mémoire et de CPU côté .NET |
| Méthodes `Async` | Identique | Threads libérés pendant l'E/S |
| `ToQueryString()` / `LogTo` | — | Vérifier ce qui part réellement en base |
