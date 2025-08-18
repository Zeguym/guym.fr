---
title: "Dotnet et SQL : bonnes pratiques pour éviter les erreurs courantes"
tags: dotnet sql best-practices
---
Voici une liste de bonnes pratiques pour éviter les erreurs courantes lors de l’utilisation de SQL dans une application .NET (que ce soit avec Entity Framework Core, Dapper ou ADO.NET).

<!--more-->

# Côté code

## 1. Pas de requêtes dans une boucle (N+1 queries)
- Mauvais : pour chaque élément d’une liste, refaire une requête DB (N+1 aller‑retours).  
  ```csharp
  foreach (var orderId in orderIds)
  {
      var order = await db.Orders.FirstAsync(o => o.Id == orderId);
      // ...
  }
  ```  
- Bon : récupérer en une seule requête, puis traiter en mémoire.  
  ```csharp
  var orders = await db.Orders
      .Where(o => orderIds.Contains(o.Id))
      .ToListAsync();
  ```

  Il est important de comprendre ce qui se passe derrière les méthodes d'accès aux données pour éviter les mauvaises surprises : une méthode `GetOrderById` qui fait une requête par ID est un anti‑pattern à éviter.

  Exemple Dapper : éviter N+1 avec un `IN`  
  ```csharp
  // Mauvais ❌ : une requête par Id
  foreach (var id in orderIds)
  {
      const string sql = "SELECT * FROM Orders WHERE Id = @Id";
      var order = await connection.QuerySingleOrDefaultAsync<Order>(sql, new { Id = id });
      // ...
  }

  // Bon ✅ : une seule requête avec un IN
  const string sqlIn = "SELECT * FROM Orders WHERE Id IN @Ids"; // syntaxe Dapper pour IN

  var ordersDictionary = (await connection.QueryAsync<Order>(sqlIn, new { Ids = orderIds }))
      .ToDictionary(o => o.Id);

  foreach (var id in orderIds)
  {
      var order = ordersDictionary[id];
      // ...
  }
  ```


## 2. Limiter le volume de données (SELECT ciblé + pagination)
- Ne pas faire `ToList()` sans filtre sur des tables volumineuses.  
- Toujours filtrer (`Where`) et paginer (`Skip` / `Take` ou `PageSize`, `PageIndex`).  
- Projeter sur un DTO avec `Select` pour ne lire que les colonnes nécessaires, surtout si les entités ont beaucoup de propriétés.

  Exemple EF Core avec pagination + projection :  
  ```csharp
  public sealed record OrderListItem(Guid Id, string Number, decimal TotalAmount);

  public async Task<IReadOnlyList<OrderListItem>> GetOrdersPageAsync(
      MyDbContext db,
      int pageIndex,
      int pageSize,
      CancellationToken cancellationToken = default)
  {
      return await db.Orders
          .Where(o => o.IsActive)
          .OrderByDescending(o => o.CreatedAt)
          .Skip(pageIndex * pageSize)
          .Take(pageSize)
          .Select(o => new OrderListItem(o.Id, o.Number, o.TotalAmount))
          .ToListAsync(cancellationToken);
  }
  ```

  Exemple Dapper avec pagination côté SQL Server :  
  ```csharp
  const string sql =
    """
    SELECT Id, Number, TotalAmount
    FROM Orders
    WHERE IsActive = 1
    ORDER BY CreatedAt DESC
    OFFSET @Offset ROWS FETCH NEXT @PageSize ROWS ONLY;
    """;

  var result = await connection.QueryAsync<OrderListItem>(
      sql,
      new { Offset = pageIndex * pageSize, PageSize = pageSize });

  return result.ToList();
  ```

## 3. Utiliser `AsNoTracking()` pour les lectures
- Pour les scénarios “read-only”, utiliser `AsNoTracking()` pour éviter le suivi de changements d’EF Core.  
- Exemple :  
  ```csharp
  var customers = await db.Customers
      .AsNoTracking()
      .Where(c => c.IsActive)
      .ToListAsync();
  ```


## 4. Toujours paramétrer les requêtes SQL
- Avec EF Core, ne jamais interpoler directement des valeurs utilisateur dans du `FromSqlRaw`. Utiliser `FromSqlInterpolated` ou des paramètres.  
- Avec Dapper/ADO.NET : toujours utiliser des paramètres (`@Id`, `SqlParameter`) et jamais de concaténation de chaînes.

  Exemple EF Core sécurisé :  
  ```csharp
  // Mauvais ❌
  var id = request.Id; // vient de l'utilisateur
  var ordersBad = await db.Orders
    .FromSqlRaw($"SELECT * FROM Orders WHERE Id = {id}")
    .ToListAsync();

  // Bon ✅
  var orders = await db.Orders
    .FromSqlInterpolated($"SELECT * FROM Orders WHERE Id = {id}")
    .ToListAsync();
  ```

  Exemple avec Dapper :  
  ```csharp
  const string sql = "SELECT * FROM Orders WHERE CustomerId = @CustomerId";

  var orders = await connection.QueryAsync<Order>(
    sql,
    new { CustomerId = customerId });
  ```

  Exemple avec ADO.NET brut :  
  ```csharp
  const string sql = "SELECT * FROM Users WHERE Email = @Email";

  using var command = new SqlCommand(sql, connection);
  command.Parameters.Add(new SqlParameter("@Email", SqlDbType.NVarChar, 256) { Value = email });

  using var reader = await command.ExecuteReaderAsync(cancellationToken);
  ```

## 5. Gestion correcte du cycle de vie (`using` / DI)
- Ne pas créer manuellement un `DbContext` avec `new` dans le code métier ; le faire injecter par DI (lifetime scoped HTTP request en ASP.NET Core).  
- Si ADO.NET direct, toujours `using var connection = new SqlConnection(...);` pour garantir la fermeture de la connexion.  
- Ne pas garder une connexion ouverte plus longtemps que nécessaire.

  En ASP.NET Core, le `DbContext` est enregistré une fois au démarrage :  
  ```csharp
  builder.Services.AddDbContext<MyDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
  ```

  Puis injecté dans les contrôleurs/services :  
  ```csharp
  public sealed class OrdersController(MyDbContext db) : ControllerBase
  {
    [HttpGet("/orders/{id:guid}")]
    public async Task<ActionResult<Order>> Get(Guid id, CancellationToken cancellationToken)
    {
      var order = await db.Orders.FindAsync(new object[] { id }, cancellationToken);
      return order is null ? NotFound() : Ok(order);
    }
  }
  ```

  Exemple ADO.NET avec `using` :  
  ```csharp
  await using var connection = new SqlConnection(connectionString);
  await connection.OpenAsync(cancellationToken);

  // ... exécuter les commandes ici ...
  ```

## 6. Async de bout en bout
- Utiliser systématiquement les variantes async (`ToListAsync`, `FirstOrDefaultAsync`, `ExecuteNonQueryAsync`, etc.).  
- Ne jamais bloquer sur du async (`.Result`, `.Wait()`) dans le code serveur : risque de deadlock.  
- Propager async jusqu’aux contrôleurs ou handlers.

  Exemple de handler/contrôleur async :  
  ```csharp
  public sealed class GetCustomerOrdersHandler(MyDbContext db)
  {
    public async Task<IReadOnlyList<Order>> HandleAsync(
      Guid customerId,
      CancellationToken cancellationToken = default)
    {
      return await db.Orders
        .AsNoTracking()
        .Where(o => o.CustomerId == customerId)
        .ToListAsync(cancellationToken);
    }
  }
  ```



## 7. Grouper les mises à jour, éviter le `SaveChanges()` dans une boucle

Appeler `SaveChanges()` à l'intérieur d'une boucle crée une transaction de base de données pour chaque itération. Cela génère :
- **Surcharge réseau** : Multiples allers-retours vers la base de données
- **Dégradation des performances** : Chaque appel à `SaveChanges()` est coûteux car il recherche les changements de toutes les entités suivies en mémoire pour chaque appel dans la boucle, génère du SQL, exécute la transaction.
- **Risques de deadlock** : Les transactions fréquentes augmentent les conflits
- **Consommation de ressources** : Ouverture/fermeture répétées de connexions


Grouper les modifications et appeler `SaveChanges()` une seule fois après la boucle :
  Exemple côté EF Core :  
  ```csharp
  foreach (var order in ordersToUpdate)
  {
    order.Status = OrderStatus.Processed;
  }

  await db.SaveChangesAsync(cancellationToken); // une seule fois
  ```

  Exemple côté Dapper avec transaction pour un batch :  
  ```csharp
  await using var connection = new SqlConnection(connectionString);
  await connection.OpenAsync(cancellationToken);

  await using var transaction = await connection.BeginTransactionAsync(cancellationToken);

  try
  {
    const string sql = "UPDATE Orders SET Status = @Status WHERE Id = @Id";

    foreach (var order in ordersToUpdate)
    {
      await connection.ExecuteAsync(
        sql,
        new { Id = order.Id, Status = OrderStatus.Processed },
        transaction);
    }

    await transaction.CommitAsync(cancellationToken);
  }
  catch
  {
    await transaction.RollbackAsync(cancellationToken);
    throw;
  }
  ```

## 8. Attention au tracking et aux mises à jour partielles
- Charger une entité pour la mettre à jour dans le même `DbContext`.  
- Pour les mises à jour partielles (patch), mapper explicitement ce qui peut être modifié pour ne pas écraser d'autres champs avec des valeurs par défaut.

Exemple (EF Core) : charger et mettre à jour dans le même contexte :  
  ```csharp
  var order = await db.Orders.FindAsync(new object[] { id }, cancellationToken);
  if (order is null)
    throw new OrderNotFoundException(id);

  order.Status = OrderStatus.Shipped;
  await db.SaveChangesAsync(cancellationToken);
  ```

  Exemple de mise à jour partielle avec mapping explicite :  
  ```csharp
  var order = await db.Orders.FindAsync(new object[] { id }, cancellationToken);
  if (order is null)
    throw new OrderNotFoundException(id);

  // Mettre à jour uniquement les champs autorisés
  if (!string.IsNullOrEmpty(request.Number))
    order.Number = request.Number;
  
  if (request.Status.HasValue)
    order.Status = request.Status.Value;

  await db.SaveChangesAsync(cancellationToken);
  ```


# Côté architecture

## 1. Ne pas mélanger logique métier et SQL brut dans le code d’appel
- Encapsuler les requêtes dans des méthodes/classe dédiées (repository / service d’accès aux données).  
- Le reste de l’application manipule des méthodes métier (`GetActiveOrdersForCustomer`, etc.), pas du SQL ou des `DbSet` partout.  
- Cela limite la duplication de requêtes complexes et facilite le refactoring.

  Exemple de service d’accès aux données (EF Core) :  
  ```csharp
  public interface IOrderReadService
  {
    Task<IReadOnlyList<Order>> GetActiveOrdersForCustomerAsync(
      Guid customerId,
      CancellationToken cancellationToken = default);
  }

  public sealed class OrderReadService(MyDbContext db) : IOrderReadService
  {
    public async Task<IReadOnlyList<Order>> GetActiveOrdersForCustomerAsync(
      Guid customerId,
      CancellationToken cancellationToken = default)
    {
      return await db.Orders
        .AsNoTracking()
        .Where(o => o.CustomerId == customerId && o.IsActive)
        .ToListAsync(cancellationToken);
    }
  }
  ```

## 2. Transactions explicites pour opérations multiples liées
- Si plusieurs changements doivent être atomiques, utiliser une transaction explicite (EF Core ou ADO.NET).  
- Garder la transaction courte et ne pas appeler des services externes (HTTP, file I/O) à l’intérieur.  
- Exemple (EF Core) :  
  ```csharp
  await using var tx = await db.Database.BeginTransactionAsync();
  // updates...
  await db.SaveChangesAsync();
  await tx.CommitAsync();
  ```

  Exemple transaction avec ADO.NET :  
  ```csharp
  await using var connection = new SqlConnection(connectionString);
  await connection.OpenAsync(cancellationToken);

  await using var transaction = await connection.BeginTransactionAsync(cancellationToken);

  try
  {
      const string insertOrderSql = "INSERT INTO Orders (Id, Number) VALUES (@Id, @Number)";
      await connection.ExecuteAsync(insertOrderSql, new { Id = id, Number = number }, transaction);

      const string insertLineSql = "INSERT INTO OrderLines (OrderId, ProductId) VALUES (@OrderId, @ProductId)";
      await connection.ExecuteAsync(insertLineSql, new { OrderId = id, ProductId = productId }, transaction);

      await transaction.CommitAsync(cancellationToken);
  }
  catch
  {
      await transaction.RollbackAsync(cancellationToken);
      throw;
  }
  ```

## 3. Gestion propre des erreurs SQL
- Toujours entourer les opérations critiques dans un `try/catch` ciblé (`DbUpdateException`, `SqlException`, etc.).  
- Loguer l’erreur, mais ne pas renvoyer les messages SQL “bruts” à l’utilisateur final.  
- Optionnel : traduire certaines erreurs en codes métier (ex : violation d’unicité → “email déjà utilisé”).

  Exemple de gestion d’erreur lors d’un `SaveChangesAsync` :  
  ```csharp
  try
  {
    await db.SaveChangesAsync(cancellationToken);
  }
  catch (DbUpdateException ex) when (ex.InnerException is SqlException sqlEx)
  {
    if (sqlEx.Number == 2601 || sqlEx.Number == 2627) // violation d'unicité
    {
      // Mapper sur une erreur métier plus propre
      throw new EmailAlreadyUsedException();
    }

    // Autres erreurs SQL => log + rethrow
    logger.LogError(ex, "Erreur lors de la mise à jour de la base");
    throw;
  }
  ```

## 4. Éviter la logique complexe dans les procédures stockées (côté .NET)

Les stored procedures peuvent sembler attrayantes pour centraliser la logique métier, mais elles présentent des inconvénients majeurs :

### Problèmes des stored procedures complexes
- **Difficiles à tester** : pas de tests unitaires simples, nécessitent une base de données.
- **Versioning complexe** : les migrations de schéma sont plus délicates.
- **Moins lisibles** : T-SQL/PL-SQL est moins expressif que C# pour la logique métier.
- **Coupling fort** : la logique métier est dispersée entre C# et la base.
- **Maintenance** : deux langages à maîtriser, deux contextes à déboguer.

### Approche recommandée
Garder SQL simple (requêtes CRUD, agrégations, filtrages) et implémenter la logique métier en C# :

Exemple : calculer le total d'une commande avec remise
```csharp
// ✅ Bon : logique métier en C#
public sealed class CalculateOrderTotalHandler(MyDbContext db)
{
  public async Task<decimal> HandleAsync(
    Guid orderId,
    CancellationToken cancellationToken = default)
  {
    var order = await db.Orders
      .Include(o => o.Lines)
      .FirstOrDefaultAsync(o => o.Id == orderId, cancellationToken);
    
    if (order is null)
      throw new OrderNotFoundException(orderId);

    var subtotal = order.Lines.Sum(l => l.Quantity * l.UnitPrice);
    var discount = subtotal >= 1000 ? subtotal * 0.1m : 0;
    
    return subtotal - discount;
  }
}
```

### Cas justifiés pour les stored procedures
- **Batch massifs** : insertion/mise à jour de millions d'enregistrements en une seule transaction.
- **Héritage système** : codebase existante qui les utilise largement.
- **Contraintes de sécurité** : permissions granulaires au niveau DB.
- **Performances critiques** : après mesure, une procédure est significativement plus rapide.

Exemple : batch d'import avec stored procedure
```csharp
const string sql = "EXEC sp_ImportOrdersFromLegacy @ImportBatchId";

await connection.ExecuteAsync(
  sql,
  new { ImportBatchId = batchId });
```



## 5. Penser aux index et au plan d’exécution
- Côté .NET il est possible de tout faire « bien », mais si les colonnes filtrées / jointes ne sont pas indexées, les performances s’écrouleront quand le volume de données augmente.  
- Vérifier régulièrement le plan d’exécution (`EXPLAIN`, `SET STATISTICS IO/TIME ON`, outil graphique du SGBD) pour les requêtes critiques.  
- Sur SQL Server, penser aux index sur les colonnes de filtrage fréquentes et sur les clés étrangères.

  Exemple simple :  
  ```sql
  -- Requête fréquemment utilisée
  SELECT *
  FROM Orders
  WHERE CustomerId = @CustomerId
    AND CreatedAt >= @FromDate;

  -- Index composite pour l’optimiser
  CREATE INDEX IX_Orders_CustomerId_CreatedAt
      ON Orders (CustomerId, CreatedAt DESC);
  ```

## 6. Timeouts raisonnables et `CancellationToken` jusqu’à la base
- Ne jamais laisser les requêtes tourner « à l’infini » : configurer des timeouts explicites (EF Core, ADO.NET).  
- Toujours propager le `CancellationToken` de la requête HTTP jusqu’aux appels SQL pour couper court quand le client est parti.

  Exemple EF Core :  
  ```csharp
  builder.Services.AddDbContext<MyDbContext>(options =>
      options.UseSqlServer(
          builder.Configuration.GetConnectionString("Default"),
          sqlOptions => sqlOptions.CommandTimeout(30) // 30 secondes
      ));
  ```

  Et dans le contrôleur/handler :  
  ```csharp
  public async Task<IActionResult> Get(CancellationToken cancellationToken)
  {
      var orders = await db.Orders
          .AsNoTracking()
          .ToListAsync(cancellationToken);

      return Ok(orders);
  }
  ```

  Exemple ADO.NET :  
  ```csharp
  using var command = new SqlCommand(sql, connection)
  {
      CommandTimeout = 30
  };

  using var reader = await command.ExecuteReaderAsync(cancellationToken);
  ```

## 7. Observer les requêtes : logs et métriques
- Activer le logging des requêtes SQL pour les environnements de dev/recette (journaliser la durée, pas seulement la requête brute).  
- Surveiller le nombre de requêtes par appel HTTP pour détecter les N+1 et les accès excessifs à la base.  
- Exposer des métriques (OpenTelemetry, Prometheus, etc.) sur la durée moyenne des requêtes DB, le nombre d’erreurs SQL, les timeouts.

  Exemple EF Core avec logging intégré :  
  ```csharp
  builder.Services.AddDbContext<MyDbContext>(options =>
      options
          .UseSqlServer(connectionString)
          .EnableSensitiveDataLogging(false) // éviter de loguer les données sensibles
          .LogTo(
              logger.LogInformation,
              new[] { DbLoggerCategory.Database.Command.Name },
              LogLevel.Information));
  ```

  # Conclusion

  Ces quinze bonnes pratiques forment une base solide pour limiter les erreurs courantes et les problèmes de performance liés à SQL dans une application .NET.

  Il n’est pas nécessaire de tout appliquer d’un coup : il est souvent plus efficace de commencer par quelques axes prioritaires, par exemple :
  - traquer les requêtes N+1 et activer les logs SQL en environnement de développement ;
  - mettre en place une pagination systématique sur les listes volumineuses ;
  - définir des timeouts raisonnables et propager correctement le `CancellationToken`.

  À partir de là, il devient plus simple d’itérer : analyser les plans d’exécution, affiner les index, renforcer l’architecture (séparation nette entre logique métier et accès aux données) puis améliorer progressivement l’observabilité et la gestion des erreurs.

