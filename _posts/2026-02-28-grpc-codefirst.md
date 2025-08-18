---
title: "gRPC Code-First en .NET : se passer des fichiers .proto"
tags: dotnet grpc code-first protobuf-net api
series: "gRPC"
---

L'approche classique de gRPC repose sur des fichiers `.proto` (Contract-First) : on définit le contrat Protobuf, puis on génère le code C# à partir de ce contrat. L'approche **Code-First** inverse le processus : on écrit des classes et interfaces C# classiques, et le contrat gRPC est généré automatiquement. Grâce à la bibliothèque **protobuf-net.Grpc**, on peut créer des services gRPC en .NET sans écrire une seule ligne de Protobuf, en utilisant des types C# natifs, des `DataContract`, ou de simples POCOs.

<!--more-->
{% include series.html %}
# Contract-First vs Code-First

## L'approche Contract-First (classique)

C'est l'approche standard de gRPC :

1. On écrit un fichier `.proto` définissant les messages et services.
2. Le compilateur `protoc` (ou les build tools .NET) génère du code C#.
3. On implémente le service en héritant de la classe de base générée.

```protobuf
// product.proto
syntax = "proto3";

service ProductService {
  rpc GetProduct (GetProductRequest) returns (Product);
}

message GetProductRequest {
  int32 id = 1;
}

message Product {
  int32 id = 1;
  string name = 2;
  double price = 3;
}
```

**Avantages** : contrat multi-langage, standard Protobuf, interop avec Go, Java, Python, etc.

**Inconvénients** : outillage `.proto`, génération de code, types générés parfois verbeux, pas de réutilisation directe des types du domaine.

## L'approche Code-First

On écrit directement en C# :

1. On définit des **interfaces** pour les services et des **classes** pour les messages.
2. On implémente le service normalement.
3. La bibliothèque `protobuf-net.Grpc` se charge de la sérialisation et du mapping gRPC.

```csharp
// Contrat partagé (C# pur)
[ServiceContract]
public interface IProductService
{
    Task<Product> GetProductAsync(GetProductRequest request);
}

[DataContract]
public class GetProductRequest
{
    [DataMember(Order = 1)]
    public int Id { get; set; }
}

[DataContract]
public class Product
{
    [DataMember(Order = 1)]
    public int Id { get; set; }

    [DataMember(Order = 2)]
    public string Name { get; set; } = "";

    [DataMember(Order = 3)]
    public double Price { get; set; }
}
```

**Avantages** : pas de `.proto`, pas de génération de code, types C# natifs, intégration naturelle avec le domaine.

**Inconvénients** : limité à l'écosystème .NET (pas d'interop directe avec d'autres langages), dépendance à `protobuf-net`.

# Mise en place

## Structure de la solution

```
Solution/
├── Shared/                    ← Contrats partagés (interfaces + DTOs)
│   └── Shared.csproj
├── ProductService/            ← Serveur gRPC
│   └── ProductService.csproj
└── ProductClient/             ← Client gRPC
    └── ProductClient.csproj
```

Le projet **Shared** contient les interfaces de service et les classes de messages. Il est référencé par le serveur et le client.

## Étape 1 : Le projet partagé (Shared)

```dotnetcli
dotnet new classlib -n Shared
cd Shared
dotnet add package protobuf-net.Grpc
```

### Définir les messages

Les messages sont de simples classes C# décorées avec `[DataContract]` et `[DataMember]` :

```csharp
using System.Runtime.Serialization;

namespace Shared;

[DataContract]
public class GetProductRequest
{
    [DataMember(Order = 1)]
    public int Id { get; set; }
}

[DataContract]
public class ProductResponse
{
    [DataMember(Order = 1)]
    public int Id { get; set; }

    [DataMember(Order = 2)]
    public string Name { get; set; } = "";

    [DataMember(Order = 3)]
    public string Description { get; set; } = "";

    [DataMember(Order = 4)]
    public decimal Price { get; set; }

    [DataMember(Order = 5)]
    public bool InStock { get; set; }
}

[DataContract]
public class ProductListResponse
{
    [DataMember(Order = 1)]
    public List<ProductResponse> Products { get; set; } = [];
}

[DataContract]
public class CreateProductRequest
{
    [DataMember(Order = 1)]
    public string Name { get; set; } = "";

    [DataMember(Order = 2)]
    public string Description { get; set; } = "";

    [DataMember(Order = 3)]
    public decimal Price { get; set; }
}
```

> **Note** : `[DataMember(Order = N)]` correspond aux identifiants de champ Protobuf. L'ordre est crucial pour la compatibilité binaire — ne jamais le changer une fois en production.

### Définir le contrat de service

```csharp
using System.ServiceModel;

namespace Shared;

[ServiceContract]
public interface IProductService
{
    [OperationContract]
    Task<ProductResponse> GetProductAsync(GetProductRequest request);

    [OperationContract]
    Task<ProductListResponse> GetAllProductsAsync();

    [OperationContract]
    Task<ProductResponse> CreateProductAsync(CreateProductRequest request);
}
```

Les attributs `[ServiceContract]` et `[OperationContract]` proviennent de `System.ServiceModel` (inclus via `protobuf-net.Grpc`). Ils ressemblent à WCF, mais ici ils servent uniquement de marqueurs pour `protobuf-net.Grpc`.

> **Remarque** : les méthodes sans paramètre (comme `GetAllProductsAsync()`) sont supportées. `protobuf-net.Grpc` gère automatiquement les appels sans message d'entrée.

## Étape 2 : Le serveur (ProductService)

```dotnetcli
dotnet new web -n ProductService
cd ProductService
dotnet add package protobuf-net.Grpc.AspNetCore
dotnet add reference ../Shared/Shared.csproj
```

### Implémenter le service

```csharp
using Shared;

namespace ProductService.Services;

public class ProductServiceImpl : IProductService
{
    private static readonly List<ProductResponse> Products =
    [
        new() { Id = 1, Name = "Clavier mécanique", Description = "Switches Cherry MX Red", Price = 89.99m, InStock = true },
        new() { Id = 2, Name = "Souris gaming", Description = "Sans fil, 25000 DPI", Price = 59.99m, InStock = true },
        new() { Id = 3, Name = "Écran 27\" 4K", Description = "IPS, 144Hz, HDR600", Price = 449.99m, InStock = false }
    ];

    private static int _nextId = 4;

    public Task<ProductResponse> GetProductAsync(GetProductRequest request)
    {
        var product = Products.FirstOrDefault(p => p.Id == request.Id);

        if (product is null)
            throw new RpcException(new Status(StatusCode.NotFound,
                $"Produit {request.Id} introuvable"));

        return Task.FromResult(product);
    }

    public Task<ProductListResponse> GetAllProductsAsync()
    {
        return Task.FromResult(new ProductListResponse
        {
            Products = Products.ToList()
        });
    }

    public Task<ProductResponse> CreateProductAsync(CreateProductRequest request)
    {
        var product = new ProductResponse
        {
            Id = _nextId++,
            Name = request.Name,
            Description = request.Description,
            Price = request.Price,
            InStock = true
        };

        Products.Add(product);
        return Task.FromResult(product);
    }
}
```

C'est une implémentation C# classique d'une interface. Pas de classe de base gRPC générée, pas de `ServerCallContext` obligatoire.

### Configurer `Program.cs`

```csharp
using ProductService.Services;
using ProtoBuf.Grpc.Server;

var builder = WebApplication.CreateBuilder(args);

// Ajouter les services gRPC code-first
builder.Services.AddCodeFirstGrpc();

var app = builder.Build();

// Mapper le service
app.MapGrpcService<ProductServiceImpl>();

app.Run();
```

On utilise `AddCodeFirstGrpc()` au lieu de `AddGrpc()`, et `MapGrpcService<T>()` fonctionne exactement de la même manière.

### Configuration Kestrel

Dans `appsettings.json`, on configure le port HTTP/2 (requis par gRPC) :

```json
{
  "Kestrel": {
    "Endpoints": {
      "Grpc": {
        "Url": "http://localhost:5001",
        "Protocols": "Http2"
      }
    }
  }
}
```

## Étape 3 : Le client (ProductClient)

```dotnetcli
dotnet new console -n ProductClient
cd ProductClient
dotnet add package Grpc.Net.Client
dotnet add package protobuf-net.Grpc
dotnet add reference ../Shared/Shared.csproj
```

### Appeler le service

```csharp
using Grpc.Net.Client;
using ProtoBuf.Grpc.Client;
using Shared;

// Créer le channel gRPC
using var channel = GrpcChannel.ForAddress("http://localhost:5001");

// Créer le client code-first à partir de l'interface
var client = channel.CreateGrpcService<IProductService>();

// Récupérer tous les produits
var allProducts = await client.GetAllProductsAsync();
Console.WriteLine("=== Tous les produits ===");
foreach (var p in allProducts.Products)
{
    Console.WriteLine($"  [{p.Id}] {p.Name} - {p.Price:C} {(p.InStock ? "✓" : "✗")}");
}

// Récupérer un produit par ID
var product = await client.GetProductAsync(new GetProductRequest { Id = 1 });
Console.WriteLine($"\nProduit #1 : {product.Name} - {product.Description}");

// Créer un nouveau produit
var created = await client.CreateProductAsync(new CreateProductRequest
{
    Name = "Webcam 4K",
    Description = "Autofocus, HDR, microphone intégré",
    Price = 129.99m
});
Console.WriteLine($"\nCréé : [{created.Id}] {created.Name} - {created.Price:C}");
```

La méthode clé est `channel.CreateGrpcService<IProductService>()` (méthode d'extension de `protobuf-net.Grpc`). Elle crée un proxy client à partir de l'interface partagée, sans code généré.

# Utiliser avec l'injection de dépendances

## Enregistrer le client dans le conteneur DI

Pour une application ASP.NET Core qui consomme un service gRPC code-first :

```dotnetcli
dotnet add package protobuf-net.Grpc.ClientFactory
```

```csharp
using ProtoBuf.Grpc.ClientFactory;
using Shared;

var builder = WebApplication.CreateBuilder(args);

// Enregistrer le client gRPC code-first via la factory
builder.Services.AddCodeFirstGrpcClient<IProductService>(options =>
{
    options.Address = new Uri("http://localhost:5001");
});

var app = builder.Build();

app.MapGet("/products", async (IProductService productService) =>
{
    var result = await productService.GetAllProductsAsync();
    return result.Products;
});

app.MapGet("/products/{id:int}", async (int id, IProductService productService) =>
{
    var product = await productService.GetProductAsync(new GetProductRequest { Id = id });
    return product;
});

app.Run();
```

`AddCodeFirstGrpcClient<IProductService>` enregistre l'interface directement dans le conteneur DI. On peut ensuite l'injecter dans n'importe quel service ou endpoint, comme n'importe quelle dépendance.

# Types supportés

`protobuf-net` supporte un large éventail de types C# natifs :

| Type C# | Protobuf équivalent |
|---------|-------------------|
| `int`, `long`, `uint`, `ulong` | `int32`, `int64`, `uint32`, `uint64` |
| `float`, `double` | `float`, `double` |
| `bool` | `bool` |
| `string` | `string` |
| `byte[]` | `bytes` |
| `decimal` | Extension protobuf-net (mappé en `bcl.Decimal`) |
| `DateTime`, `DateTimeOffset` | Extension protobuf-net |
| `TimeSpan` | Extension protobuf-net |
| `Guid` | Extension protobuf-net |
| `List<T>`, `T[]` | `repeated` |
| `Dictionary<TKey, TValue>` | `map` |
| types `nullable` (`int?`, etc.) | Champs optionnels |
| `enum` | `enum` Protobuf |

> **Attention** : les types spécifiques .NET (`decimal`, `DateTime`, `Guid`, etc.) utilisent des extensions `protobuf-net` qui ne sont **pas interopérables** avec des clients dans d'autres langages. Si vous avez besoin d'interop multi-langage, restez sur les types de base.

# Streaming

`protobuf-net.Grpc` supporte les 4 modes de streaming gRPC via `IAsyncEnumerable<T>` :

## Server Streaming

```csharp
// Contrat
[ServiceContract]
public interface IProductService
{
    [OperationContract]
    IAsyncEnumerable<ProductResponse> StreamAllProductsAsync(
        CancellationToken cancellationToken = default);
}
```

```csharp
// Implémentation serveur
public async IAsyncEnumerable<ProductResponse> StreamAllProductsAsync(
    [EnumeratorCancellation] CancellationToken cancellationToken)
{
    foreach (var product in Products)
    {
        cancellationToken.ThrowIfCancellationRequested();
        yield return product;
        await Task.Delay(500, cancellationToken); // Simule un traitement
    }
}
```

```csharp
// Client
await foreach (var product in client.StreamAllProductsAsync())
{
    Console.WriteLine($"Reçu : {product.Name}");
}
```

## Client Streaming

```csharp
// Contrat
[ServiceContract]
public interface IProductService
{
    [OperationContract]
    Task<ImportSummary> ImportProductsAsync(
        IAsyncEnumerable<CreateProductRequest> products,
        CancellationToken cancellationToken = default);
}
```

```csharp
// Client
async IAsyncEnumerable<CreateProductRequest> GenerateProducts()
{
    yield return new CreateProductRequest { Name = "Produit A", Price = 10.0m };
    yield return new CreateProductRequest { Name = "Produit B", Price = 20.0m };
    yield return new CreateProductRequest { Name = "Produit C", Price = 30.0m };
}

var summary = await client.ImportProductsAsync(GenerateProducts());
Console.WriteLine($"Importés : {summary.Count}");
```

L'utilisation de `IAsyncEnumerable<T>` est idiomatique en C# et bien plus naturelle que les `IServerStreamWriter<T>` / `IAsyncStreamReader<T>` de l'approche contract-first.

# Accéder au `CallContext`

Si vous avez besoin d'accéder aux metadata gRPC, aux headers ou au `CancellationToken` côté serveur, vous pouvez ajouter un paramètre `CallContext` :

```csharp
using ProtoBuf.Grpc;

[ServiceContract]
public interface IProductService
{
    [OperationContract]
    Task<ProductResponse> GetProductAsync(
        GetProductRequest request,
        CallContext context = default);
}
```

```csharp
// Implémentation serveur
public Task<ProductResponse> GetProductAsync(
    GetProductRequest request,
    CallContext context)
{
    // Accéder aux headers de la requête
    var authHeader = context.RequestHeaders?.GetValue("authorization");

    // Accéder au ServerCallContext natif gRPC
    var serverContext = context.ServerCallContext;
    var peer = serverContext?.Peer;

    // ...
    return Task.FromResult(product);
}
```

Le `CallContext` est optionnel (`= default`), donc les clients qui ne le fournissent pas fonctionnent normalement.

# Combiner avec Dapr

L'approche code-first fonctionne aussi avec le proxy gRPC de Dapr. Le client pointe vers le sidecar au lieu du service directement :

```csharp
var daprGrpcPort = Environment.GetEnvironmentVariable("DAPR_GRPC_PORT") ?? "50001";

builder.Services.AddCodeFirstGrpcClient<IProductService>(options =>
{
    options.Address = new Uri($"http://localhost:{daprGrpcPort}");
})
.AddCallCredentials((context, metadata) =>
{
    metadata.Add("dapr-app-id", "product-service");
    return Task.CompletedTask;
});
```

Le sidecar Dapr route l'appel vers le bon service, avec découverte automatique, mTLS et résilience intégrés.

# Générer le `.proto` à partir du code

Si vous avez besoin du fichier `.proto` (pour de la documentation ou de l'interop avec d'autres langages), `protobuf-net` peut le générer à partir de vos types C# :

```csharp
using ProtoBuf.Grpc.Reflection;

var generator = new SchemaGenerator();
var schema = generator.GetSchema<IProductService>();

File.WriteAllText("product.proto", schema);
Console.WriteLine(schema);
```

Cela produit un fichier `.proto` standard qui peut être utilisé par des clients Go, Java, Python, etc.

# Contract-First vs Code-First : quand choisir quoi ?

| Critère | Contract-First (`.proto`) | Code-First (`protobuf-net`) |
|---------|--------------------------|---------------------------|
| **Interop multi-langage** | Natif | Limité (.NET uniquement, sauf export `.proto`) |
| **Génération de code** | Obligatoire (protoc/build) | Aucune |
| **Types supportés** | Types Protobuf de base | Types C# natifs (`decimal`, `DateTime`, `Guid`…) |
| **Streaming** | `IServerStreamWriter`, `IAsyncStreamReader` | `IAsyncEnumerable<T>` (idiomatique C#) |
| **Courbe d'apprentissage** | Syntaxe `.proto` + outillage | C# pur, attributs familiers |
| **WCF migration** | Réécriture | Transition naturelle (`[ServiceContract]`, `[DataContract]`) |
| **Écosystème** | Standard gRPC | Communauté `protobuf-net` |
| **Performance** | Optimale | Comparable (même transport gRPC/HTTP/2) |

### Choisir Contract-First quand :

- Les clients sont dans **plusieurs langages** (Go, Java, Python…).
- Vous voulez le **standard Protobuf** pur.
- Vous travaillez dans un écosystème gRPC existant.

### Choisir Code-First quand :

- Tous les clients et serveurs sont en **.NET**.
- Vous voulez partager les **types du domaine** directement.
- Vous migrez depuis **WCF**.
- Vous préférez éviter la complexité des fichiers `.proto` et de la génération de code.
- Vous voulez utiliser des types C# riches (`decimal`, `DateTime`, `IAsyncEnumerable`).

# Résumé

| Aspect | Détail |
|--------|--------|
| **Bibliothèque** | `protobuf-net.Grpc` (serveur : `.AspNetCore`, client : `.ClientFactory`) |
| **Contrat** | Interface C# avec `[ServiceContract]` + classes `[DataContract]` |
| **Pas de `.proto`** | Le mapping Protobuf est déduit des attributs C# |
| **Streaming** | Via `IAsyncEnumerable<T>`, idiomatique en C# |
| **DI** | `AddCodeFirstGrpc()` côté serveur, `AddCodeFirstGrpcClient<T>()` côté client |
| **Interop** | Export `.proto` possible via `SchemaGenerator` |
| **Compatible Dapr** | Oui, via le proxy gRPC du sidecar |

L'approche Code-First de gRPC offre une expérience 100% C#, sans fichier `.proto` ni génération de code, tout en conservant les performances de gRPC/HTTP/2. C'est un excellent choix pour les projets .NET-only qui veulent la performance de gRPC avec la simplicité du développement C# classique.
