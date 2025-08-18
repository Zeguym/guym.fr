---
title: l’inversion de contrôle et injection de dépendance en dotnet (Ioc 1/3)
tags: dotnet ioc injection-dependance
series: "IoC et DI en .NET"
---

**IoC (Inversion of Control)** est un **principe de conception** utilisé pour réduire le couplage entre les composants d'une application. Il consiste à inverser la responsabilité de la création et de la gestion des dépendances d'une classe, en déléguant cette responsabilité à un conteneur ou à un framework.

<!--more-->
{% include series.html %}
## 1. Principe de l'IoC

Dans une application classique, une classe crée directement ses dépendances. Avec IoC, la classe ne crée pas ses dépendances : elles lui sont fournies par un conteneur ou un framework. Cela permet de rendre le code plus flexible, modulaire et testable.

### Exemple sans IoC :

```csharp
public class OrderService
{
    private readonly ILogger _logger;

    public OrderService()
    {
        _logger = new ConsoleLogger(); // La classe crée directement sa dépendance
    }

    public void ProcessOrder()
    {
        _logger.Log("Processing order...");
    }
}
```

### Exemple avec IoC :

```csharp
public class OrderService
{
    private readonly ILogger _logger;

    public OrderService(ILogger logger) // La dépendance est injectée
    {
        _logger = logger;
    }

    public void ProcessOrder()
    {
        _logger.Log("Processing order...");
    }
}
```

Cet exemple illustre qu’il suffit de modifier la configuration du conteneur pour remplacer le logger dans tout le projet, sans avoir à changer chaque instance dans le code.

## IoC et Dependency Injection (DI)

L'IoC est un concept général, tandis que la **Dependency Injection (DI)** est une implémentation spécifique de l'IoC. Avec la DI, les dépendances sont injectées dans une classe par un conteneur IoC.

## Conteneur IoC

Un **conteneur IoC** est un outil qui gère la création, la configuration et la durée de vie des objets (dépendances). En .NET, le conteneur IoC natif est intégré dans le framework, mais des bibliothèques tierces comme **Autofac**, **Ninject**, ou **Castle Windsor** peuvent également être utilisées.

## \*Avantages de l'IoC

- **Réduction du couplage** : Les classes dépendent d'abstractions (interfaces) plutôt que d'implémentations concrètes.
- **Facilité de test** : Les dépendances peuvent être remplacées par des mocks dans les tests unitaires.
- **Réutilisabilité** : Les classes deviennent plus modulaires et réutilisables.
- **Gestion centralisée** : Les dépendances sont configurées à un seul endroit.

## IoC dans .NET

En .NET, l'IoC est intégré via le conteneur d'injection de dépendance natif. Les services sont enregistrés dans le conteneur (`IServiceCollection`) et injectés automatiquement dans les classes qui en ont besoin.

### Exemple :

```csharp
var builder = WebApplication.CreateBuilder(args);

// Enregistrement des services
builder.Services.AddTransient<ILogger, ConsoleLogger>();
builder.Services.AddScoped<IOrderService, OrderService>();

var app = builder.Build();
```

L'**IoC** est un principe fondamental pour écrire du code modulaire, testable et maintenable. Il est au cœur de nombreux frameworks modernes, y compris .NET.

---

# L'injection de dépendance en .NET

L'injection de dépendance (Dependency Injection, DI) est un **design pattern** qui permet de gérer les dépendances entre les classes de manière flexible et modulaire. En .NET, ce concept est intégré directement dans le framework, ce qui facilite son utilisation.

## 1. Qu'est-ce qu'une dépendance ?

Une dépendance est un objet dont une classe a besoin pour fonctionner. Par exemple, si une classe `OrderService` utilise une classe `ILogger` pour enregistrer des logs, alors `ILogger` est une dépendance de `OrderService`.

### Exemple sans injection de dépendance :

```csharp
public class OrderService
{
    private readonly ILogger _logger;

    public OrderService()
    {
        _logger = new ConsoleLogger(); // La dépendance est instanciée directement ici
    }

    public void ProcessOrder()
    {
        _logger.Log("Processing order...");
    }
}
```

Dans cet exemple, `OrderService` est **couplé** à `ConsoleLogger`. Si vous voulez changer le type de logger, vous devez modifier le code de `OrderService`.

## 2. Pourquoi utiliser l'injection de dépendance ?

- **Réduction du couplage** : Les classes ne dépendent pas directement des implémentations concrètes, mais des abstractions (interfaces).
- **Facilité de test** : Les dépendances peuvent être remplacées par des mocks ou des stubs dans les tests unitaires.
- **Réutilisabilité** : Les classes deviennent plus modulaires et réutilisables.
- **Gestion centralisée des dépendances** : Les dépendances sont configurées à un seul endroit.

## 3. Types d'injection de dépendance (Natif dotnet core)

Il existe trois types principaux d'injection de dépendance :

### a) Injection par constructeur

C'est la méthode la plus courante. Les dépendances sont passées via le constructeur.

```csharp
public class OrderService
{
    private readonly ILogger _logger;

    public OrderService(ILogger logger)
    {
        _logger = logger;
    }

    public void ProcessOrder()
    {
        _logger.Log("Processing order...");
    }
}
```

### b) Injection par propriété (Lib tierces)

Les dépendances sont injectées via des propriétés publiques.

```csharp
public class OrderService
{
    public ILogger Logger { get; set; }

    public void ProcessOrder()
    {
        Logger?.Log("Processing order...");
    }
}
```

### c) Injection par méthode (Partiel dans donet core)

Les dépendances sont passées en tant que paramètres de méthode.

```csharp
public class OrderService
{
    public void ProcessOrder(ILogger logger)
    {
        logger.Log("Processing order...");
    }
}
```

---

# Injection de dépendance native en .NET

Depuis .NET Core, Microsoft fournit un conteneur d'injection de dépendance intégré, simple et performant. Ce conteneur est conçu pour répondre aux besoins courants des développeurs, tout en restant léger et facile à utiliser.

## 1. Fonctionnement de l'injection native

### a) Enregistrement des services

Les dépendances sont enregistrées dans le conteneur via la méthode `IServiceCollection` dans `Program.cs` ou `Startup.cs`. Vous pouvez spécifier le cycle de vie des services :

- **`AddTransient<TService, TImplementation>()`** : Crée une nouvelle instance à chaque fois que le service est demandé.
- **`AddScoped<TService, TImplementation>()`** : Crée une instance par requête HTTP (ou par portée dans les applications non-web).
- **`AddSingleton<TService, TImplementation>()`** : Crée une seule instance pour toute la durée de vie de l'application.

### Exemple :

```csharp
var builder = WebApplication.CreateBuilder(args);

// Enregistrement des services
builder.Services.AddTransient<ILogger, ConsoleLogger>(); // Transient
builder.Services.AddScoped<IOrderService, OrderService>(); // Scoped
builder.Services.AddSingleton<IConfiguration>(builder.Configuration); // Singleton

var app = builder.Build();
```

### b) Résolution des dépendances

Les dépendances sont injectées automatiquement dans les classes qui en ont besoin. Cela peut se faire :

- Par **constructeur** (le plus courant natif dotnet).
- Par **propriété** (moins recommandé).
- Par **méthode** (rarement utilisé).

Exemple avec injection par constructeur :

```csharp
public class OrderService : IOrderService
{
    private readonly ILogger _logger;

    public OrderService(ILogger logger)
    {
        _logger = logger;
    }

    public void ProcessOrder()
    {
        _logger.Log("Order processed successfully.");
    }
}
```

### c) Utilisation dans les contrôleurs ou middleware

Dans une application ASP.NET Core, les dépendances sont injectées automatiquement dans les contrôleurs ou middleware.

Exemple dans un contrôleur :

```csharp
[ApiController]
[Route("[controller]")]
public class OrderController : ControllerBase
{
    private readonly IOrderService _orderService;

    public OrderController(IOrderService orderService)
    {
        _orderService = orderService;
    }

    [HttpGet]
    public IActionResult ProcessOrder()
    {
        _orderService.ProcessOrder();
        return Ok("Order processed!");
    }
}
```

## 2. Injection de dépendance par attribut dans les contrôleurs et Minimal API

En .NET, il est possible d'injecter des dépendances directement dans les contrôleurs ou les handlers des Minimal APIs en utilisant des **attributs** comme `[FromServices]`. Cela permet d'injecter des services spécifiques sans passer par le constructeur.

### Injection par attribut dans les contrôleurs

Dans les contrôleurs ASP.NET Core, vous pouvez utiliser l'attribut `[FromServices]` pour injecter une dépendance directement dans une méthode d'action. Cela est utile si vous ne voulez pas surcharger le constructeur avec trop de dépendances.

#### Exemple :

```csharp
[ApiController]
[Route("[controller]")]
public class OrderController : ControllerBase
{
    [HttpGet]
    public IActionResult ProcessOrder([FromServices] IOrderService orderService)
    {
        orderService.ProcessOrder();
        return Ok("Order processed!");
    }
}
```

#### Points importants :

- L'attribut `[FromServices]` indique au framework d'injecter la dépendance depuis le conteneur de services.
- Cela peut être utile pour des dépendances rarement utilisées ou spécifiques à une méthode.

### Injection par attribut dans les Minimal APIs

Dans les Minimal APIs, vous pouvez également injecter des dépendances directement dans les handlers en utilisant des paramètres. Le framework injecte automatiquement les services enregistrés dans le conteneur.

#### Exemple :

```csharp
var builder = WebApplication.CreateBuilder(args);

// Enregistrement des services
builder.Services.AddTransient<IOrderService, OrderService>();

var app = builder.Build();

app.MapGet("/process-order", (IOrderService orderService) =>
{
    orderService.ProcessOrder();
    return Results.Ok("Order processed!");
});

app.Run();
```

#### Points importants :

- Dans les Minimal APIs, il n'est pas nécessaire d'utiliser `[FromServices]`. Le framework injecte automatiquement les dépendances si elles sont enregistrées dans le conteneur.
- Les paramètres du handler sont résolus automatiquement par le conteneur de services.

### Comparaison entre injection par constructeur et par attribut

| **Aspect**                    | **Injection par constructeur**                       | **Injection par attribut**                      |
| ----------------------------- | ---------------------------------------------------- | ----------------------------------------------- |
| **Usage principal**           | Pour les dépendances utilisées dans toute la classe. | Pour les dépendances spécifiques à une méthode. |
| **Surcharge du constructeur** | Peut rendre le constructeur trop complexe.           | Évite de surcharger le constructeur.            |
| **Testabilité**               | Plus facile à tester avec des mocks.                 | Moins courant dans les tests unitaires.         |
| **Lisibilité**                | Centralise les dépendances dans le constructeur.     | Peut rendre le code moins lisible si abusé.     |

### Bonnes pratiques

- **Privilégiez l'injection par constructeur** pour les dépendances utilisées dans toute la classe.
- Utilisez **l'injection par attribut** pour des dépendances spécifiques à une méthode ou rarement utilisées.
- Évitez d'abuser de l'injection par attribut, car cela peut rendre le code plus difficile à maintenir.

## 3. Injection de dépendance par clé en .NET

Depuis **.NET 8**, le conteneur d'injection de dépendance natif prend en charge l'injection **par clé**. Cela permet d'enregistrer plusieurs implémentations d'une même interface et de choisir dynamiquement laquelle utiliser en fonction d'une clé.

### Pourquoi utiliser l'injection par clé ?

L'injection par clé est utile lorsque :

- Vous avez plusieurs implémentations d'une même interface.
- Vous devez choisir dynamiquement l'implémentation à utiliser en fonction d'un contexte ou d'une clé.

### Enregistrement des services par clé

Pour enregistrer des services par clé, utilisez la méthode **`AddKeyedService`**. Vous associez chaque implémentation à une clé unique.

#### Exemple :

```csharp
var builder = WebApplication.CreateBuilder(args);

// Enregistrement des services avec des clés
builder.Services.AddKeyedService<IOrderService, StandardOrderService>("Standard");
builder.Services.AddKeyedService<IOrderService, PremiumOrderService>("Premium");

var app = builder.Build();
```

Dans cet exemple :

- `StandardOrderService` est associé à la clé `"Standard"`.
- `PremiumOrderService` est associé à la clé `"Premium"`.

### Résolution des services par clé

Pour résoudre un service par clé, vous pouvez injecter un **`IKeyedServiceProvider`** dans votre classe ou méthode. Cela vous permet de récupérer l'implémentation associée à une clé spécifique.

#### Exemple :

```csharp
public class OrderProcessor
{
    private readonly IKeyedServiceProvider<IOrderService> _orderServiceProvider;

    public OrderProcessor(IKeyedServiceProvider<IOrderService> orderServiceProvider)
    {
        _orderServiceProvider = orderServiceProvider;
    }

    public void ProcessOrder(string orderType)
    {
        var orderService = _orderServiceProvider.GetRequiredService(orderType);
        orderService.ProcessOrder();
    }
}
```

Dans cet exemple :

- `GetRequiredService(orderType)` retourne l'implémentation associée à la clé `orderType`.

### Utilisation dans Minimal APIs\*

Dans les Minimal APIs, vous pouvez également utiliser l'injection par clé pour choisir dynamiquement un service.

#### Exemple :

```csharp
app.MapGet("/process-order/{type}", (string type, IKeyedServiceProvider<IOrderService> serviceProvider) =>
{
    var orderService = serviceProvider.GetRequiredService(type);
    orderService.ProcessOrder();
    return Results.Ok($"Order processed with {type} service!");
});
```

### Comparaison avec d'autres approches

| **Aspect**            | **Injection par clé**                               | **Injection classique**                          |
| --------------------- | --------------------------------------------------- | ------------------------------------------------ |
| **Flexibilité**       | Permet de choisir dynamiquement une implémentation. | Une seule implémentation est injectée.           |
| **Complexité**        | Ajoute une couche de complexité.                    | Plus simple à configurer et à utiliser.          |
| **Cas d'utilisation** | Scénarios avec plusieurs implémentations.           | Scénarios simples avec une seule implémentation. |

### Bonnes pratiques

- Utilisez des clés descriptives pour éviter toute confusion.
- Documentez les clés et leurs implémentations associées.
- Évitez d'utiliser l'injection par clé si une seule implémentation est nécessaire.

---

# Points supplémentaires à savoir sur l'injection de dépendance en .NET

## 1. Résolution des dépendances optionnelles

Vous pouvez injecter des dépendances facultatives en utilisant le type nullable ou en vérifiant si une dépendance est `null`.

### Exemple :

```csharp
public class OrderService
{
    private readonly ILogger? _logger;

    public OrderService(ILogger? logger = null)
    {
        _logger = logger;
    }

    public void ProcessOrder()
    {
        _logger?.Log("Processing order...");
    }
}
```

Cela permet de ne pas enregistrer certaines dépendances dans le conteneur si elles ne sont pas toujours nécessaires.

## 2. Résolution des collections de services

Vous pouvez enregistrer plusieurs implémentations d'une même interface et les injecter sous forme de collection.

### Exemple :

```csharp
var builder = WebApplication.CreateBuilder(args);

// Enregistrement de plusieurs implémentations
builder.Services.AddTransient<IOrderService, StandardOrderService>();
builder.Services.AddTransient<IOrderService, PremiumOrderService>();

var app = builder.Build();
```

Injection dans une classe :

```csharp
public class OrderProcessor
{
    private readonly IEnumerable<IOrderService> _orderServices;

    public OrderProcessor(IEnumerable<IOrderService> orderServices)
    {
        _orderServices = orderServices;
    }

    public void ProcessAllOrders()
    {
        foreach (var service in _orderServices)
        {
            service.ProcessOrder();
        }
    }
}
```

## 3. Injection dans des services non gérés par le conteneur

Si vous devez injecter des dépendances dans des classes non gérées par le conteneur (ex. : classes statiques ou créées manuellement), vous pouvez utiliser un **`IServiceProvider`**.

### Exemple :

```csharp
var serviceProvider = builder.Services.BuildServiceProvider();
var orderService = serviceProvider.GetRequiredService<IOrderService>();
orderService.ProcessOrder();
```

## 4. Gestion des dépendances circulaires

Les dépendances circulaires (A dépend de B, et B dépend de A) peuvent provoquer des erreurs. Pour les éviter :

- Refactorisez votre code pour réduire les dépendances.
- Utilisez des abstractions (interfaces) pour briser les cycles.

## 5. Utilisation avancée avec des scopes

Les scopes permettent de gérer des dépendances spécifiques à une requête ou un contexte.

### Exemple :

```csharp
app.MapGet("/scoped", async (IServiceScopeFactory scopeFactory) =>
{
    using var scope = scopeFactory.CreateScope();
    var service = scope.ServiceProvider.GetRequiredService<IOrderService>();
    service.ProcessOrder();
});
```

## 6. Tests unitaires avec l'injection de dépendance

L'injection de dépendance facilite les tests unitaires en permettant de remplacer les dépendances par des mocks.

### Exemple avec Moq :

```csharp
var mockLogger = new Mock<ILogger>();
var service = new OrderService(mockLogger.Object);

service.ProcessOrder();

mockLogger.Verify(l => l.Log(It.IsAny<string>()), Times.Once);
```

---

# Limitations de l'injection native

Bien que l'injection de dépendance native en .NET soit puissante, elle présente certaines limitations :

### a) Fonctionnalités limitées

Le conteneur natif est volontairement minimaliste. Il ne prend pas en charge certaines fonctionnalités avancées que proposent des bibliothèques tierces :

- **Résolution conditionnelle** : Pas de support natif pour enregistrer plusieurs implémentations d'une interface et choisir dynamiquement laquelle utiliser.
- **Injection basée sur des noms ou des clés** : Impossible de différencier les services enregistrés par des identifiants. (dispo depuis .net8)
- **Interception ou décoration** : Pas de support direct pour intercepter les appels (ex. : ajout de logique avant/après une méthode).

### b) Pas de gestion avancée des cycles de vie

Le conteneur natif ne permet pas de gérer des cycles de vie complexes ou personnalisés (ex. : un service qui doit être recréé après un certain temps).

### c) Pas de support pour les modules

Contrairement à des bibliothèques comme Autofac, il n'y a pas de notion de "modules" pour organiser les enregistrements de dépendances dans des blocs logiques.

### d) Pas de diagnostic avancé

Le conteneur natif ne fournit pas d'outils avancés pour diagnostiquer les problèmes liés à la résolution des dépendances (ex. : dépendances circulaires, services manquants).

### e) Pas de support pour les scopes imbriqués

Dans certaines applications complexes, vous pourriez avoir besoin de scopes imbriqués pour gérer des dépendances spécifiques à une sous-partie d'une requête. Le conteneur natif ne gère pas cela directement.

---

## Quand utiliser des bibliothèques tierces ?

Si les limitations ci-dessus deviennent un problème dans votre projet, vous pouvez envisager d'utiliser des bibliothèques tierces comme :

- **Autofac** : Fournit des fonctionnalités avancées comme les modules, les résolutions conditionnelles, et les décorateurs.
- **Ninject** : Offre une syntaxe fluide pour configurer les dépendances et des fonctionnalités avancées.
- **Castle Windsor** : Très puissant pour les scénarios complexes.
- **Simple Injector** : Conçu pour la performance et la simplicité.

Ces bibliothèques peuvent être intégrées dans une application .NET en remplaçant le conteneur natif.

---

## Exemple avec Autofac (remplacement du conteneur natif)

### Installation

Ajoutez le package NuGet :

```bash
dotnet add package Autofac.Extensions.DependencyInjection
```

### Configuration

```csharp
var builder = WebApplication.CreateBuilder(args);

// Remplacement du conteneur natif par Autofac
builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory());
builder.Host.ConfigureContainer<ContainerBuilder>(containerBuilder =>
{
    containerBuilder.RegisterType<ConsoleLogger>().As<ILogger>().InstancePerDependency();
    containerBuilder.RegisterType<OrderService>().As<IOrderService>().InstancePerLifetimeScope();
});

var app = builder.Build();
```

# Conclusion

L'injection de dépendance native en .NET est suffisante pour la très grande majorité des applications. Elle est :

- Simple à configurer.
- Intégrée directement dans le framework.
- Performante pour les scénarios courants.

Cependant, pour des besoins avancés (résolution conditionnelle, interception, diagnostics), il peut être utile de se tourner vers des bibliothèques tierces comme Autofac ou Ninject.
