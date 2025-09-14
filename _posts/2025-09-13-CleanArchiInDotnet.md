---
title: Clean Architecture en .NET
tags: dotnet clean-architecture 
---

La Clean Architecture a pour objectif de structurer une application de façon à séparer clairement les responsabilités et à rendre le cœur métier indépendant des détails techniques (frameworks, base de données, UI, etc.). Elle organise le code en couches concentriques, où les règles métier ne dépendent jamais des couches externes, ce qui facilite les tests, l’évolution et la maintenance.

En isolant les dépendances externes derrière des interfaces et en contrôlant le sens des dépendances (toujours vers le domaine), on obtient un système plus robuste, modulaire et durable, capable d’absorber les changements technologiques sans remettre en cause le cœur fonctionnel.

<!--more-->

## 1. Objectifs de la Clean Architecture

### Indépendance des frameworks

- Les frameworks (comme ASP.NET Core ou Entity Framework) ne doivent pas dicter la structure de votre application.
- Les règles métier doivent être isolées des détails techniques pour éviter un couplage fort.

### Testabilité

- Les couches internes (comme les entités et les cas d'utilisation) doivent pouvoir être testées sans dépendre des couches externes.
- Cela permet d'écrire des tests unitaires rapides et fiables.

### Indépendance de l'interface utilisateur

- Vous devez pouvoir remplacer l'interface utilisateur (ex. : passer d'une API REST à une interface graphique) sans modifier les règles métier.

### Indépendance des bases de données

- Le choix de la base de données (SQL Server, MongoDB, etc.) ne doit pas affecter les règles métier.
- Cela permet de changer de technologie de stockage sans réécrire la logique métier.

## 2. Structure de la Clean Architecture (détaillée)

### Couches principales

1. Entities (Entités) :

   - Contient les règles métier fondamentales.
   - Ces règles sont indépendantes de toute technologie ou framework.
   - Exemple : Une classe `Product` avec des validations métier.

2. Use Cases (Cas d'utilisation) :

   - Définit les actions spécifiques que l'application peut effectuer.
   - Ces cas d'utilisation orchestrent les interactions entre les entités et les services externes.
   - Exemple : Un service `UpdateProductPrice` qui met à jour le prix d'un produit.

3. Interface Adapters (Adaptateurs d'interface) :

   - Contient les implémentations concrètes pour interagir avec les couches externes.
   - Exemple : Contrôleurs API, DTOs, mappers.

4. Frameworks & Drivers (Cadres et pilotes) :
   - Contient les détails techniques (frameworks, bibliothèques tierces).
   - Exemple : ASP.NET Core pour l'interface utilisateur, Entity Framework pour la base de données.

## 3. Règle clé : l'inversion de dépendance (détaillée)

### Principe fondamental

- Les dépendances doivent toujours pointer **vers l'intérieur**.
- Les couches internes (Use Cases, Entities) ne doivent jamais dépendre des couches externes (UI, base de données, infrastructure).

### Comment l'appliquer ?

- Utilisez des **interfaces** pour découpler les couches.
- Les couches internes définissent les interfaces, et les couches externes fournissent les implémentations.

### Exemple

```csharp
// Couche interne (Use Case)
public interface IProductRepository
{
    Product GetById(int id);
    void Save(Product product);
}

// Couche externe (Infrastructure)
public class ProductRepository : IProductRepository
{
    private readonly DbContext _context;

    public ProductRepository(DbContext context)
    {
        _context = context;
    }

    public Product GetById(int id)
    {
        return _context.Products.Find(id);
    }

    public void Save(Product product)
    {
        _context.Products.Update(product);
        _context.SaveChanges();
    }
}
```

## 4. Exemple d'implémentation détaillée

### Couche Entités

Les entités contiennent les règles métier fondamentales.

```csharp
namespace MyApp.Core.Entities
{
    public class Product
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public decimal Price { get; set; }

        public void ApplyDiscount(decimal percentage)
        {
            if (percentage < 0 || percentage > 100)
                throw new ArgumentException("Invalid discount percentage");

            Price -= Price * (percentage / 100);
        }
    }
}
```

### Couche Use Cases

Les cas d'utilisation orchestrent les interactions entre les entités et les services externes.

```csharp
namespace MyApp.Core.UseCases
{
    public class UpdateProductPrice
    {
        private readonly IProductRepository _repository;

        public UpdateProductPrice(IProductRepository repository)
        {
            _repository = repository;
        }

        public void Execute(int productId, decimal newPrice)
        {
            var product = _repository.GetById(productId);
            product.Price = newPrice;
            _repository.Save(product);
        }
    }
}
```

### Couche Interface Adapters

Les adaptateurs d'interface exposent les cas d'utilisation via des contrôleurs ou des DTOs.

```csharp
namespace MyApp.Web.Controllers
{
    [ApiController]
    [Route("api/products")]
    public class ProductsController : ControllerBase
    {
        private readonly UpdateProductPrice _updateProductPrice;

        public ProductsController(UpdateProductPrice updateProductPrice)
        {
            _updateProductPrice = updateProductPrice;
        }

        [HttpPut("{id}")]
        public IActionResult UpdatePrice(int id, [FromBody] decimal newPrice)
        {
            _updateProductPrice.Execute(id, newPrice);
            return Ok();
        }
    }
}
```

### Couche Frameworks & Drivers

Les frameworks et bibliothèques tierces sont isolés dans cette couche.

```csharp
namespace MyApp.Infrastructure.Persistence
{
    public class ProductRepository : IProductRepository
    {
        private readonly DbContext _context;

        public ProductRepository(DbContext context)
        {
            _context = context;
        }

        public Product GetById(int id)
        {
            return _context.Products.Find(id);
        }

        public void Save(Product product)
        {
            _context.Products.Update(product);
            _context.SaveChanges();
        }
    }
}
```

## 5. Avantages détaillés

### Séparation des préoccupations

- Chaque couche a une responsabilité claire, ce qui facilite la maintenance.

### Testabilité

- Les couches internes (Use Cases, Entities) peuvent être testées indépendamment des couches externes.

### Flexibilité

- Vous pouvez remplacer les couches externes (ex. : passer d'Entity Framework à Dapper) sans modifier les règles métier.

### Évolutivité

- Le code est structuré de manière à faciliter l'ajout de nouvelles fonctionnalités.

## 6. Bonnes pratiques détaillées

1. Respecter le sens des dépendances :

    - Les couches internes (domaine, cas d'utilisation) ne doivent jamais dépendre directement des couches externes (UI, base de données, infrastructure).

2. Utiliser des abstractions :

   - Les interfaces permettent de découpler les couches.

3. Limiter les responsabilités :

   - Chaque couche doit se concentrer sur son rôle spécifique.

4. Isoler les frameworks :

   - Les frameworks doivent être confinés dans la couche "Frameworks & Drivers".

5. Tester les couches internes :
   - Les tests unitaires doivent se concentrer sur les couches `Entities` et `Use Cases`.
