---
title: Entity Framework Part 4
tags: dotnet efcore
---


# Les bonne pratiques à appliquer

## 1. Privilégier les requêtes côté serveur

-   Utilise des méthodes LINQ qui se traduisent en SQL (`Where`, `Select`, `OrderBy`, etc.).
-   Évite d’appeler des méthodes .NET non traduites en SQL avant d’avoir récupéré les données (ex : `.ToList()`, `.AsEnumerable()`).

## 2. Utiliser le filtrage le plus tôt possible

-   Place les `.Where()` avant les `.Select()` ou `.OrderBy()` pour limiter le volume de données transférées.

## 3. Projections avec `Select`

-   Utilise `.Select()` pour ne récupérer que les colonnes nécessaires (DTO, ViewModel).
-   Exemple :

```csharp
var clients = db.Clients    .Where(c => c.IsActive)    .Select(c => new ClientDto { Id = c.Id, Name = c.Name })      .ToList();
```

## 4. Pagination

-   Utilise `.Skip()` et `.Take()` pour paginer côté SQL.

```csharp
var page = db.Products      .OrderBy(p => p.Name)      .Skip(pageIndex * pageSize)      .Take(pageSize)      .ToList();
```

## 5. Chargement des relations

-   Privilégier `.Include()` pour charger explicitement les relations nécessaires.

```csharp
var orders = db.Orders.Include(o => o.Customer).ToList();
```

-   Éviter le chargement paresseux (lazy loading) si tu veux contrôler les requêtes.

## 6. Éviter les requêtes N+1

Le problème N+1 survient quand tu fais une requête pour une entité principale, puis une requête supplémentaire pour chaque entité liée.

-   Utilise `.Include()` ou des projections pour éviter de charger les entités liées une par une.

### Exemple du problème N+1

Supposons que tu veux récupérer une liste de commandes et, pour chaque commande, ses lignes :

```csharp
// Provoque N+1 requêtes SQL !
var commandes = db.CdeClients.ToList();
foreach (var commande in commandes)
{    
    var lignes = db.LigCdeClients.Where(l => l.CdeClientId == commande.Id).ToList();    // ...
}
```
-   Ici, 1 requête pour les commandes, puis 1 requête par commande pour les lignes.

### Bonne pratique : charger les relations en une seule requête

Utilise `.Include()` pour charger les entités liées en une seule requête SQL :

```csharp
var commandes = db.CdeClients    .Include(c => c.Lignes) // navigation property    .ToList();
```

-   Résultat : 1 seule requête SQL qui ramène les commandes **et** leurs lignes.

### Alternative avec projection

Pour un DTO il est possble de projeter comme ceci :

```csharp
var commandes = db.CdeClients .Select(c => new CommandeDto  
{       
    Id = c.Id,  
    Lignes = c.LigCdeClients.Select(l => new LigneDto { Id = l.Id, ... }).ToList() 
})
.ToList();
```
Ici, Entity Framework génèrera une seule requête SQL avec jointure.

## 7. **Utiliser `Any()` et `Count()` judicieusement**

-   Préfère `Any()` pour vérifier l’existence, plus performant que `Count() > 0`.

```csharp
bool hasOrders = db.Orders.Any(o => o.CustomerId == id);
```

Mais attention aux any et count qui génèrent des sous requêtes (N+1 requêtes)

```csharp
 db.Customers.Where(c => c.Orders.Any(o => o.IsPaid));
```

Un select customer de select top 1 order vas être généré une jointure avec les commandes payés sera plus efficace.

## 8. **Utiliser `AsNoTracking()` pour la lecture**

-   Pour les requêtes en lecture seule, ajoute `.AsNoTracking()` pour améliorer les performances.

```csharp
var products = db.Products.AsNoTracking().ToList();
```

## 9. **Éviter les requêtes complexes côté client**

-   Pas de jointures ou de transformations complexes après `.ToList()`. Garde le maximum côté SQL.

## 10. **Utiliser les expressions lambda claires**

-   Privilégier des expressions lisibles et nomme les variables de façon explicite.

## 11. **les methodes asynchrones**

Il est recommandé d’utiliser les méthodes asynchrones (`async`) avec Entity Framework pour plusieurs raisons :

- **Performance et scalabilité** : Les méthodes async (`ToListAsync`, `SaveChangesAsync`, etc.) libèrent le thread pendant l’attente des opérations I/O (accès à la base), permettant au serveur de traiter d’autres requêtes.  
- **Réactivité** : Dans les applications web ou desktop, l’utilisation d’async évite de bloquer l’interface utilisateur ou les threads du serveur.
- **Bonne pratique moderne** : Le modèle asynchrone est adapté aux architectures cloud et microservices, où la gestion efficace des ressources est essentielle.

Utiliser les méthodes async avec Entity Framework améliore la performance, la réactivité et la capacité à traiter de nombreuses requêtes simultanément.