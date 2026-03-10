---
title: "Dapr : le Pub/Sub (Publish & Subscribe) en .NET"
tags: dotnet dapr microservices pub-sub messaging
series: "Dapr pour les développeurs .NET"
---

Le Pub/Sub (Publish & Subscribe) est l'un des building blocks les plus utilisés de Dapr. Il permet à des services de communiquer de manière **asynchrone** via des événements, sans couplage direct entre l'émetteur et le(s) récepteur(s). En .NET, le SDK Dapr et ASP.NET Core rendent la publication et la souscription d'événements simples et déclaratives, quel que soit le broker sous-jacent (RabbitMQ, Kafka, Azure Service Bus, Redis Streams…).

<!--more-->
{% include series.html %}

# Le problème

Dans une architecture microservices, les services ont souvent besoin de se notifier mutuellement sans s'appeler directement :

- **Découplage** : le service qui émet un événement ne devrait pas connaître (ni dépendre de) ceux qui le consomment.
- **Scalabilité** : plusieurs consommateurs doivent pouvoir traiter le même événement indépendamment.
- **Fiabilité** : les messages doivent être livrés au moins une fois (at-least-once), même si un consommateur est temporairement indisponible.
- **Portabilité** : on veut pouvoir changer de broker (passer de RabbitMQ à Kafka, par exemple) sans modifier le code applicatif.
- **Format d'événements** : chaque broker a ses propres SDK, formats de messages, modes de sérialisation.

Sans Dapr, il faut intégrer le SDK du broker choisi, gérer la connexion, la sérialisation, les retries, les dead-letter queues, et coupler le code à un fournisseur spécifique. Dapr résout tout cela avec une API Pub/Sub unifiée.

# Fonctionnement

Le Pub/Sub de Dapr repose sur le modèle **publish/subscribe** classique, enrichi par le format **CloudEvents** :

1. Un service **publie** un événement sur un **topic** via son sidecar Dapr.
2. Le sidecar transmet le message au **broker** configuré (RabbitMQ, Kafka, etc.).
3. Les sidecars des services **abonnés** récupèrent le message et le transmettent à leur application via un appel HTTP ou gRPC sur `localhost`.

```mermaid
graph LR
    PubApp["Service A<br/>(Publisher)"]
    PubSidecar["Sidecar A"]
    Broker["Broker<br/>(RabbitMQ, Kafka,<br/>Azure Service Bus...)"]
    SubSidecar1["Sidecar B"]
    SubApp1["Service B<br/>(Subscriber)"]
    SubSidecar2["Sidecar C"]
    SubApp2["Service C<br/>(Subscriber)"]
    
    PubApp -->|PublishEvent<br/>localhost| PubSidecar
    PubSidecar -->|Publie| Broker
    Broker -->|Distribue| SubSidecar1
    Broker -->|Distribue| SubSidecar2
    SubSidecar1 -->|POST /endpoint<br/>localhost| SubApp1
    SubSidecar2 -->|POST /endpoint<br/>localhost| SubApp2
    
    style PubApp fill:#4A90E2
    style PubSidecar fill:#F5A623
    style Broker fill:#D0021B
    style SubSidecar1 fill:#F5A623
    style SubSidecar2 fill:#F5A623
    style SubApp1 fill:#7ED321
    style SubApp2 fill:#7ED321
```

Chaque message est automatiquement enveloppé au format **CloudEvents** (standard CNCF), ce qui garantit un format de métadonnées interopérable.

# Configuration du composant

Le composant Pub/Sub est défini dans un fichier YAML. Voici quelques exemples :

## Redis Streams (développement local)

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.redis
  version: v1
  metadata:
    - name: redisHost
      value: "localhost:6379"
    - name: redisPassword
      value: ""
```

## RabbitMQ

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
    - name: connectionString
      value: "amqp://guest:guest@localhost:5672"
    - name: durable
      value: "true"
    - name: deletedWhenUnused
      value: "false"
```

## Apache Kafka

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: "localhost:9092"
    - name: consumerGroup
      value: "my-consumer-group"
    - name: authType
      value: "none"
```

## Azure Service Bus

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.azure.servicebus.topics
  version: v1
  metadata:
    - name: connectionString
      value: "Endpoint=sb://my-namespace.servicebus.windows.net/;SharedAccessKeyName=...;SharedAccessKey=..."
```

Le point essentiel : **le code applicatif est identique** quel que soit le broker. On passe de Redis à Kafka en modifiant uniquement le fichier YAML.

# Pub/Sub en .NET

## Installation

```dotnetcli
dotnet add package Dapr.AspNetCore
```

## Configuration du service

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDaprClient();

var app = builder.Build();

// Activer le middleware CloudEvents (désérialisation automatique)
app.UseCloudEvents();

// Enregistrer les souscriptions auprès du sidecar
app.MapSubscribeHandler();

app.Run();
```

`UseCloudEvents()` permet à ASP.NET Core de désérialiser automatiquement les enveloppes CloudEvents pour extraire le payload. `MapSubscribeHandler()` expose un endpoint `/dapr/subscribe` que le sidecar interroge au démarrage pour connaître les topics auxquels l'application est abonnée.

# Publier des événements

## Avec `DaprClient`

```csharp
public class OrderService
{
    private readonly DaprClient _daprClient;
    private const string PubSubName = "pubsub";

    public OrderService(DaprClient daprClient)
    {
        _daprClient = daprClient;
    }

    public async Task CreateOrderAsync(Order order)
    {
        // Logique de création de commande...
        order.Status = "Created";

        // Publier l'événement sur le topic "orders"
        await _daprClient.PublishEventAsync(PubSubName, "orders", new OrderCreated
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            TotalAmount = order.TotalAmount,
            CreatedAt = DateTime.UtcNow
        });
    }
}
```

## Publication depuis un endpoint Minimal API

```csharp
app.MapPost("/orders", async (Order order, DaprClient dapr) =>
{
    order.Id = Guid.NewGuid().ToString();
    order.Status = "Created";

    // Sauvegarder la commande (state store)
    await dapr.SaveStateAsync("statestore", $"order-{order.Id}", order);

    // Publier l'événement
    await dapr.PublishEventAsync("pubsub", "orders", new OrderCreated
    {
        OrderId = order.Id,
        CustomerId = order.CustomerId,
        TotalAmount = order.TotalAmount,
        CreatedAt = DateTime.UtcNow
    });

    return Results.Created($"/orders/{order.Id}", order);
});
```

## Publication avec métadonnées

On peut ajouter des métadonnées au message (headers personnalisés, TTL, etc.) :

```csharp
var metadata = new Dictionary<string, string>
{
    { "ttlInSeconds", "600" },      // Le message expire après 10 minutes
    { "rawPayload", "false" }       // Utiliser le format CloudEvents (par défaut)
};

await _daprClient.PublishEventAsync("pubsub", "orders", orderCreated, metadata);
```

# S'abonner aux événements

## Souscription déclarative avec l'attribut `[Topic]`

La manière la plus simple de s'abonner à un topic en ASP.NET Core :

```csharp
app.MapPost("/order-created", [Topic("pubsub", "orders")] (OrderCreated evt) =>
{
    Console.WriteLine($"Commande reçue : {evt.OrderId}, montant : {evt.TotalAmount}€");
    // Traitement de l'événement...
    return Results.Ok();
});
```

L'attribut `[Topic("pubsub", "orders")]` indique au sidecar Dapr que cet endpoint doit recevoir les messages publiés sur le topic `orders` du composant `pubsub`.

## Souscription avec un Controller

```csharp
[ApiController]
[Route("[controller]")]
public class NotificationsController : ControllerBase
{
    private readonly ILogger<NotificationsController> _logger;

    public NotificationsController(ILogger<NotificationsController> logger)
    {
        _logger = logger;
    }

    [Topic("pubsub", "orders")]
    [HttpPost("order-created")]
    public async Task<IActionResult> HandleOrderCreated(OrderCreated evt)
    {
        _logger.LogInformation("Notification pour commande {OrderId}", evt.OrderId);

        // Envoyer un email, une notification push, etc.
        await SendNotificationAsync(evt);

        return Ok();
    }

    private Task SendNotificationAsync(OrderCreated evt)
    {
        // Logique de notification...
        return Task.CompletedTask;
    }
}
```

## Souscription par fichier de configuration (déclarative externe)

On peut aussi déclarer les souscriptions dans un fichier YAML, sans modifier le code :

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: order-subscription
spec:
  pubsubname: pubsub
  topic: orders
  routes:
    default: /order-created
  scopes:
    - notification-service
    - inventory-service
```

Cette approche est utile quand on veut gérer les souscriptions de manière centralisée, ou quand le service cible ne peut pas être modifié.

# CloudEvents

Dapr utilise le format **CloudEvents** (spécification CNCF) pour standardiser les messages. Chaque message publié est automatiquement encapsulé dans une enveloppe CloudEvents :

```json
{
  "specversion": "1.0",
  "type": "com.dapr.event.sent",
  "source": "order-service",
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "datacontenttype": "application/json",
  "data": {
    "orderId": "order-42",
    "customerId": "cust-123",
    "totalAmount": 99.99,
    "createdAt": "2026-03-07T10:30:00Z"
  }
}
```

| Champ | Description |
|-------|-------------|
| `specversion` | Version de la spec CloudEvents |
| `type` | Type de l'événement |
| `source` | App-id du service émetteur |
| `id` | Identifiant unique du message |
| `datacontenttype` | Type MIME du payload |
| `data` | Le contenu de l'événement (votre objet sérialisé) |

Grâce à `UseCloudEvents()`, le middleware ASP.NET Core extrait automatiquement le champ `data` et le désérialise dans le type attendu par l'endpoint.

## Publication en raw (sans CloudEvents)

Si un consommateur ne comprend pas le format CloudEvents, on peut publier en mode « raw » :

```csharp
var metadata = new Dictionary<string, string>
{
    { "rawPayload", "true" }
};

await _daprClient.PublishEventAsync("pubsub", "orders", orderCreated, metadata);
```

Le message sera publié tel quel, sans enveloppe CloudEvents.

# Sémantique de livraison

## At-least-once (par défaut)

Dapr garantit une livraison **au moins une fois** : si le subscriber ne retourne pas un succès (200 OK), le sidecar re-tente la livraison. Cela implique que le handler doit être **idempotent** (capable de traiter le même message plusieurs fois sans effet de bord).

### Stratégies d'idempotence

```csharp
app.MapPost("/order-created", [Topic("pubsub", "orders")] async (
    OrderCreated evt, DaprClient dapr) =>
{
    // Vérifier si l'événement a déjà été traité (idempotence)
    var alreadyProcessed = await dapr.GetStateAsync<bool>(
        "statestore", $"processed-{evt.OrderId}");

    if (alreadyProcessed)
    {
        // Déjà traité, on retourne OK pour acquitter le message
        return Results.Ok();
    }

    // Traiter l'événement...
    await ProcessOrderAsync(evt);

    // Marquer comme traité
    await dapr.SaveStateAsync("statestore", $"processed-{evt.OrderId}", true);

    return Results.Ok();
});
```

## Contrôle de la réponse du subscriber

Le sidecar interprète la réponse HTTP du subscriber pour décider du sort du message :

| Code de réponse | Action du sidecar |
|----------------|-------------------|
| `200` (OK) | Message acquitté (ACK), supprimé de la queue |
| `404` (Not Found) | Message abandonné (DROP), supprimé de la queue |
| `Autre` (5xx, timeout…) | Message re-tenté (RETRY) selon la politique de résilience |

On peut aussi retourner un statut explicite via le body :

```csharp
app.MapPost("/order-created", [Topic("pubsub", "orders")] (OrderCreated evt) =>
{
    try
    {
        ProcessOrder(evt);
        return Results.Ok(new { status = "SUCCESS" });
    }
    catch (InvalidOperationException)
    {
        // Message invalide, ne pas réessayer
        return Results.Ok(new { status = "DROP" });
    }
    catch (Exception)
    {
        // Erreur temporaire, réessayer
        return Results.StatusCode(500);
    }
});
```

# Dead Letter Topics

Quand un message ne peut pas être traité après plusieurs tentatives, il est envoyé dans un **dead letter topic** pour analyse ultérieure. La configuration se fait dans le composant ou dans la souscription :

## Configuration dans la souscription

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: order-subscription
spec:
  pubsubname: pubsub
  topic: orders
  routes:
    default: /order-created
  deadLetterTopic: orders-deadletter
```

## Consommer les dead letters

```csharp
app.MapPost("/dead-letters", [Topic("pubsub", "orders-deadletter")] (
    CloudEvent<OrderCreated> deadLetter) =>
{
    Console.WriteLine($"Message en échec : {deadLetter.Data?.OrderId}");
    // Logger, alerter, stocker pour investigation...
    return Results.Ok();
});
```

# Routing d'événements

Dapr permet de router les messages vers différents endpoints selon leur contenu, grâce aux **règles de routage** :

## Configuration via souscription YAML

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: order-subscription
spec:
  pubsubname: pubsub
  topic: orders
  routes:
    rules:
      - match: event.data.status == "created"
        path: /orders/created
      - match: event.data.status == "shipped"
        path: /orders/shipped
      - match: event.data.status == "cancelled"
        path: /orders/cancelled
    default: /orders/unknown
```

## Configuration dans le code avec l'attribut `[TopicRule]` (ASP.NET Core)

On peut également exprimer les règles de routage directement dans le code via l'attribut `[Topic]` enrichi :

```csharp
// Route par défaut
app.MapPost("/orders/default",
    [Topic("pubsub", "orders", isDefault: true)]
    (OrderEvent evt) =>
{
    Console.WriteLine($"Événement ordre non classifié : {evt.OrderId}");
    return Results.Ok();
});

// Route pour les créations
app.MapPost("/orders/created",
    [Topic("pubsub", "orders", "event.data.status == \"created\"", 1)]
    (OrderEvent evt) =>
{
    Console.WriteLine($"Commande créée : {evt.OrderId}");
    return Results.Ok();
});

// Route pour les expéditions
app.MapPost("/orders/shipped",
    [Topic("pubsub", "orders", "event.data.status == \"shipped\"", 2)]
    (OrderEvent evt) =>
{
    Console.WriteLine($"Commande expédiée : {evt.OrderId}");
    return Results.Ok();
});
```

# Scoping : restreindre l'accès aux topics

Dapr permet de restreindre quels services peuvent publier ou s'abonner à quels topics :

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
    - name: connectionString
      value: "amqp://guest:guest@localhost:5672"
  scopes:
    - order-service
    - notification-service
    - inventory-service
```

On peut affiner avec des règles de publication et de souscription :

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: order-sub
spec:
  pubsubname: pubsub
  topic: orders
  routes:
    default: /order-created
  scopes:
    - notification-service   # Seul ce service peut s'abonner
```

# Résilience

Comme pour les autres building blocks, Dapr permet de configurer des politiques de résilience sur le Pub/Sub :

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: resiliency
spec:
  policies:
    retries:
      pubsubRetry:
        policy: constant
        duration: 2s
        maxRetries: 5
    circuitBreakers:
      pubsubBreaker:
        maxRequests: 1
        interval: 30s
        timeout: 60s
        trip: consecutiveFailures > 3
  targets:
    components:
      pubsub:
        outbound:
          retry: pubsubRetry
          circuitBreaker: pubsubBreaker
        inbound:
          retry: pubsubRetry
```

| Cible | Description |
|-------|-------------|
| `outbound` | Politique appliquée lors de la **publication** |
| `inbound` | Politique appliquée lors de la **livraison** au subscriber |

# Bulk Publish et Bulk Subscribe

Pour les cas à haut débit, Dapr supporte la publication et la souscription en lots (bulk).

## Bulk Publish

Publier plusieurs événements en une seule requête, réduisant les allers-retours réseau :

```csharp
public async Task PublishOrdersBatchAsync(IEnumerable<OrderCreated> events)
{
    var bulkEvents = events.Select(e =>
        new BulkPublishEntry<OrderCreated>(
            entryId: Guid.NewGuid().ToString(),
            @event: e,
            contentType: "application/json")).ToList();

    var response = await _daprClient.BulkPublishEventAsync(
        "pubsub", "orders", bulkEvents);

    if (response.FailedEntries.Count > 0)
    {
        Console.WriteLine($"{response.FailedEntries.Count} messages en échec.");
        foreach (var failed in response.FailedEntries)
        {
            Console.WriteLine($"  - {failed.Entry.EntryId} : {failed.ErrorMessage}");
        }
    }
}
```

## Bulk Subscribe

Recevoir plusieurs messages en un seul appel HTTP pour améliorer le débit de consommation :

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: bulk-order-subscription
spec:
  pubsubname: pubsub
  topic: orders
  routes:
    default: /orders/bulk
  bulkSubscribe:
    enabled: true
    maxMessagesCount: 100
    maxAwaitDurationMs: 1000
```

```csharp
app.MapPost("/orders/bulk", (BulkSubscribeMessage<OrderCreated> bulkMessage) =>
{
    var statuses = new List<BulkSubscribeAppResponseEntry>();

    foreach (var entry in bulkMessage.Entries)
    {
        try
        {
            ProcessOrder(entry.Event);
            statuses.Add(new BulkSubscribeAppResponseEntry(
                entry.EntryId, BulkSubscribeAppResponseStatus.SUCCESS));
        }
        catch (Exception)
        {
            statuses.Add(new BulkSubscribeAppResponseEntry(
                entry.EntryId, BulkSubscribeAppResponseStatus.RETRY));
        }
    }

    return Results.Ok(new BulkSubscribeAppResponse(statuses));
});
```

Chaque message du lot peut être acquitté ou rejeté individuellement.

# API HTTP sous-jacente

## Publier un événement

```http
POST http://localhost:3500/v1.0/publish/pubsub/orders
Content-Type: application/json

{
  "orderId": "order-42",
  "customerId": "cust-123",
  "totalAmount": 99.99,
  "createdAt": "2026-03-07T10:30:00Z"
}
```

## Lister les souscriptions

```http
GET http://localhost:3500/v1.0/metadata
```

## Bulk Publish

```http
POST http://localhost:3500/v1.0-alpha1/publish/bulk/pubsub/orders
Content-Type: application/json

{
  "entries": [
    {
      "entryId": "1",
      "event": { "orderId": "order-1", "totalAmount": 50 },
      "contentType": "application/json"
    },
    {
      "entryId": "2",
      "event": { "orderId": "order-2", "totalAmount": 75 },
      "contentType": "application/json"
    }
  ]
}
```

# Exemple complet : architecture événementielle

Voici un exemple concret avec trois services communiquant par Pub/Sub :

## Le publisher : OrderService

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDaprClient();
var app = builder.Build();

app.MapPost("/orders", async (CreateOrderRequest request, DaprClient dapr) =>
{
    var order = new Order
    {
        Id = Guid.NewGuid().ToString(),
        CustomerId = request.CustomerId,
        Items = request.Items,
        TotalAmount = request.Items.Sum(i => i.Price * i.Quantity),
        Status = "Created",
        CreatedAt = DateTime.UtcNow
    };

    // Sauvegarder la commande
    await dapr.SaveStateAsync("statestore", $"order-{order.Id}", order);

    // Publier l'événement
    await dapr.PublishEventAsync("pubsub", "orders", new OrderCreated
    {
        OrderId = order.Id,
        CustomerId = order.CustomerId,
        Items = order.Items,
        TotalAmount = order.TotalAmount,
        CreatedAt = order.CreatedAt
    });

    return Results.Created($"/orders/{order.Id}", order);
});

app.Run();
```

## Le subscriber 1 : InventoryService

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDaprClient();
var app = builder.Build();

app.UseCloudEvents();
app.MapSubscribeHandler();

app.MapPost("/order-created",
    [Topic("pubsub", "orders")] async (OrderCreated evt, DaprClient dapr) =>
{
    foreach (var item in evt.Items)
    {
        // Décrémenter le stock
        var stock = await dapr.GetStateAsync<int>("statestore", $"stock-{item.ProductId}");
        stock -= item.Quantity;

        if (stock < 0)
        {
            // Publier un événement de rupture de stock
            await dapr.PublishEventAsync("pubsub", "stock-alerts", new StockAlert
            {
                ProductId = item.ProductId,
                CurrentStock = stock,
                OrderId = evt.OrderId
            });
        }

        await dapr.SaveStateAsync("statestore", $"stock-{item.ProductId}", stock);
    }

    return Results.Ok();
});

app.Run();
```

## Le subscriber 2 : NotificationService

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDaprClient();
var app = builder.Build();

app.UseCloudEvents();
app.MapSubscribeHandler();

// Notification de commande
app.MapPost("/order-created",
    [Topic("pubsub", "orders")] async (OrderCreated evt) =>
{
    Console.WriteLine($"📧 Email envoyé au client {evt.CustomerId} " +
                      $"pour la commande {evt.OrderId} ({evt.TotalAmount}€)");

    await SendEmailAsync(evt.CustomerId,
        $"Votre commande {evt.OrderId} a été confirmée !");

    return Results.Ok();
});

// Alerte de stock
app.MapPost("/stock-alert",
    [Topic("pubsub", "stock-alerts")] async (StockAlert alert) =>
{
    Console.WriteLine($"⚠️ Alerte stock : produit {alert.ProductId}, " +
                      $"stock actuel : {alert.CurrentStock}");

    await NotifyAdminAsync(alert);

    return Results.Ok();
});

app.Run();

async Task SendEmailAsync(string customerId, string message) =>
    await Task.Delay(100); // Simulation

async Task NotifyAdminAsync(StockAlert alert) =>
    await Task.Delay(100); // Simulation
```

## Les modèles partagés

```csharp
public record OrderCreated
{
    public string OrderId { get; init; } = string.Empty;
    public string CustomerId { get; init; } = string.Empty;
    public List<OrderItem> Items { get; init; } = [];
    public decimal TotalAmount { get; init; }
    public DateTime CreatedAt { get; init; }
}

public record OrderItem
{
    public int ProductId { get; init; }
    public string Name { get; init; } = string.Empty;
    public decimal Price { get; init; }
    public int Quantity { get; init; }
}

public record StockAlert
{
    public int ProductId { get; init; }
    public int CurrentStock { get; init; }
    public string OrderId { get; init; } = string.Empty;
}
```

# Lancement en local

```bash
# Terminal 1 : OrderService (publisher)
dapr run --app-id order-service --app-port 5000 \
    --resources-path ./components -- dotnet run --project OrderService

# Terminal 2 : InventoryService (subscriber)
dapr run --app-id inventory-service --app-port 5001 \
    --resources-path ./components -- dotnet run --project InventoryService

# Terminal 3 : NotificationService (subscriber)
dapr run --app-id notification-service --app-port 5002 \
    --resources-path ./components -- dotnet run --project NotificationService
```

## Tester la publication

```bash
curl -X POST http://localhost:5000/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "cust-123",
    "items": [
      { "productId": 1, "name": "Clavier", "price": 49.99, "quantity": 1 },
      { "productId": 2, "name": "Souris", "price": 29.99, "quantity": 2 }
    ]
  }'
```

Les deux subscribers reçoivent automatiquement l'événement `OrderCreated` publié par `OrderService`.

# Lancement avec .NET Aspire

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var pubSub = builder.AddDaprPubSub("pubsub");
var stateStore = builder.AddDaprStateStore("statestore");

builder.AddProject<Projects.OrderService>("order-service")
    .WithDaprSidecar()
    .WithReference(pubSub)
    .WithReference(stateStore);

builder.AddProject<Projects.InventoryService>("inventory-service")
    .WithDaprSidecar()
    .WithReference(pubSub)
    .WithReference(stateStore);

builder.AddProject<Projects.NotificationService>("notification-service")
    .WithDaprSidecar()
    .WithReference(pubSub);

builder.Build().Run();
```

# Bonnes pratiques

| Pratique | Pourquoi |
|----------|----------|
| **Handlers idempotents** | Le mode at-least-once peut livrer un message plusieurs fois |
| **Événements fins et spécifiques** | Préférer `OrderCreated`, `OrderShipped` à un générique `OrderUpdated` |
| **Schéma versionné** | Ajouter un champ `version` ou `type` dans les événements pour gérer l'évolution |
| **Dead letter topics** | Toujours configurer un dead letter topic pour éviter de perdre des messages |
| **Scoping** | Restreindre l'accès aux topics aux seuls services concernés |
| **TTL sur les messages** | Éviter l'accumulation de messages obsolètes dans le broker |
| **Monitoring** | Surveiller les métriques Dapr (messages publiés, acquittés, rejetés) via les dashboards |

# Résumé

| Aspect | Détail |
|--------|--------|
| **API** | `POST http://localhost:3500/v1.0/publish/{pubsub-name}/{topic}` |
| **Format** | CloudEvents (standard CNCF) ou raw payload |
| **Livraison** | At-least-once par défaut |
| **Souscription** | Attribut `[Topic]` en code, ou fichier YAML externe |
| **Routing** | Règles de routage par contenu de l'événement |
| **Dead letters** | Topic de messages en échec, configurable |
| **Bulk** | Publication et souscription par lots pour le haut débit |
| **Résilience** | Retry, circuit breaker configurables en YAML |
| **Scoping** | Restriction d'accès par service |
| **Brokers** | Redis, RabbitMQ, Kafka, Azure Service Bus, Pulsar… |
| **SDK .NET** | `DaprClient.PublishEventAsync`, attribut `[Topic]`, `UseCloudEvents()` |

Le Pub/Sub Dapr offre une abstraction puissante et portable de la communication asynchrone entre services, avec des garanties de livraison, du routage conditionnel et une gestion des erreurs intégrée, tout en permettant de changer de broker sans modifier une seule ligne de code applicatif.
