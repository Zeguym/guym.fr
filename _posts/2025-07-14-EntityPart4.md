---
title: Bonnes pratiques pour les requêtes (EFcore)
tags: dotnet efcore clean-queries 
series: "Entity Framework"
---

Entity Framework Core : bonnes pratiques pour écrire des requêtes performantes et éviter les pièges courants

<!--more-->
 {% include series.html %}
# Bonnes pratiques à appliquer

## 1. Privilégier les requêtes côté serveur

- Utiliser des méthodes LINQ qui se traduisent en SQL (`Where`, `Select`, `OrderBy`, etc.).
- Éviter d’appeler des méthodes .NET non traduites en SQL avant d’avoir récupéré les données (par exemple : `.ToList()`, `.AsEnumerable()`).

## 2. Appliquer le filtrage le plus tôt possible

- Placer les `.Where()` avant les `.Select()` ou `.OrderBy()` afin de limiter le volume de données transférées.

## 3. Projections avec `Select`

- Utiliser `.Select()` pour ne récupérer que les colonnes nécessaires (DTO, ViewModel).
- Exemple :

```csharp
var clients = db.Clients    .Where(c => c.IsActive)    .Select(c => new ClientDto { Id = c.Id, Name = c.Name })      .ToList();
```

## 4. Pagination

- Utiliser `.Skip()` et `.Take()` pour effectuer la pagination côté SQL.

```csharp
var page = db.Products      .OrderBy(p => p.Name)      .Skip(pageIndex * pageSize)      .Take(pageSize)      .ToList();
```

## 5. Chargement des relations

- Privilégier l’utilisation de `.Include()` pour charger explicitement les relations nécessaires.

```csharp
var orders = db.Orders.Include(o => o.Customer).ToList();
```

- Éviter le chargement paresseux (lazy loading) lorsque l’on souhaite garder un contrôle précis sur les requêtes exécutées.

## 6. Éviter les requêtes N+1

Le problème N+1 survient lorsqu’une requête est exécutée pour une entité principale, puis une requête supplémentaire pour chaque entité liée.

- Utiliser `.Include()` ou des projections pour éviter de charger les entités liées une par une.

### Exemple du problème N+1

Supposons que l’on souhaite récupérer une liste de commandes et, pour chaque commande, ses lignes :

```csharp
// Provoque N+1 requêtes SQL !
var commandes = db.CdeClients.ToList();
foreach (var commande in commandes)
{
    var lignes = db.LigCdeClients.Where(l => l.CdeClientId == commande.Id).ToList();    // ...
}
```

- Ici, 1 requête pour les commandes, puis 1 requête par commande pour les lignes.

### Bonne pratique : charger les relations en une seule requête

Utiliser `.Include()` pour charger les entités liées en une seule requête SQL :

```csharp
var commandes = db.CdeClients    .Include(c => c.Lignes) // navigation property    .ToList();
```

- Résultat : 1 seule requête SQL qui ramène les commandes **et** leurs lignes.

### Alternative avec projection

Pour un DTO, il est possible de projeter les données de la manière suivante :

```csharp
var commandes = db.CdeClients .Select(c => new CommandeDto
{
    Id = c.Id,
    Lignes = c.LigCdeClients.Select(l => new LigneDto { Id = l.Id, ... }).ToList()
})
.ToList();
```

Dans ce cas, Entity Framework générera une seule requête SQL avec jointure.

## 7. Utiliser `Any()` et `Count()` de manière appropriée

- Préférer `Any()` pour vérifier l’existence, généralement plus performant que `Count() > 0`.

```csharp
bool hasOrders = db.Orders.Any(o => o.CustomerId == id);
```

Il convient toutefois de rester attentif aux utilisations de `Any()` et `Count()` susceptibles de générer des sous-requêtes (et donc des requêtes de type N+1).

```csharp
 db.Customers.Where(c => c.Orders.Any(o => o.IsPaid));
```

Dans certains cas, une requête explicite avec jointure, par exemple en sélectionnant le client et en filtrant sur les commandes payées, sera plus efficace qu’une sous-requête répétée.

## 8. Utiliser `AsNoTracking()` pour la lecture

- Pour les requêtes en lecture seule, ajouter `.AsNoTracking()` permet d’améliorer les performances.

```csharp
var products = db.Products.AsNoTracking().ToList();
```

## 9. Éviter les requêtes complexes côté client

- Éviter les jointures ou les transformations complexes après l’appel à `.ToList()`. Il est préférable de conserver le maximum de logique côté SQL.

## 10. Utiliser des expressions lambda claires

- Privilégier des expressions lisibles et nommer les variables de manière explicite.

## 11. Utiliser les méthodes asynchrones

Il est recommandé d’utiliser les méthodes asynchrones (`async`) avec Entity Framework pour plusieurs raisons :

- **Performance et scalabilité** : les méthodes asynchrones (`ToListAsync`, `SaveChangesAsync`, etc.) libèrent le thread pendant l’attente des opérations d’E/S (accès à la base), ce qui permet au serveur de traiter d’autres requêtes.
- **Réactivité** : dans les applications web ou desktop, l’utilisation d’`async` évite de bloquer l’interface utilisateur ou les threads du serveur.
- **Bonne pratique moderne** : le modèle asynchrone est particulièrement adapté aux architectures cloud et microservices, où la gestion efficace des ressources est essentielle.

L’utilisation systématique des méthodes asynchrones avec Entity Framework contribue à améliorer les performances, la réactivité et la capacité à traiter un grand nombre de requêtes simultanément.
