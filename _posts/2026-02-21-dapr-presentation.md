---
title: "Présentation de Dapr : le runtime pour applications distribuées"
tags: dotnet dapr microservices architecture
series: "Dapr pour les développeurs .NET"
---

Dapr (Distributed Application Runtime) est un runtime open source, porté par Microsoft et incubé à la CNCF, qui simplifie le développement d'applications distribuées (microservices, applications cloud-native). Il fournit un ensemble de **building blocks** — invocation de services, gestion d'état, pub/sub, bindings, secrets, etc. — accessibles via des API HTTP ou gRPC, quel que soit le langage ou le framework utilisé. L'objectif : permettre aux développeurs de se concentrer sur la logique métier, sans se noyer dans la plomberie de l'infrastructure.

<!--more-->
{% include series.html %}

# Pourquoi Dapr ?

Quand on développe des microservices, on se retrouve rapidement face à des problèmes récurrents :

- **Communication inter-services** : comment appeler un autre service de façon fiable, avec retry, circuit breaker, etc. ?
- **Gestion d'état** : comment stocker et retrouver l'état d'un service de manière distribuée ?
- **Pub/Sub** : comment publier et consommer des événements entre services ?
- **Secrets** : comment accéder aux secrets de manière sécurisée ?
- **Observabilité** : comment tracer les appels entre services ?

Chaque problème a ses propres bibliothèques, SDK, et configurations spécifiques selon le broker, la base de données, le fournisseur de secrets, etc. Dapr propose une **couche d'abstraction unifiée** qui résout tous ces problèmes via des API standardisées, indépendantes de l'implémentation sous-jacente.

# Architecture de Dapr

Dapr fonctionne en **sidecar** : un processus léger (`daprd`) tourne à côté de chaque instance de votre application. Votre code communique avec le sidecar via HTTP ou gRPC sur `localhost`, et c'est le sidecar qui gère toute la complexité (réseau, retry, sérialisation, connexion au broker, etc.).

```mermaid
graph LR
    App["Application<br/>(.NET, Go, Python...)"]
    Sidecar["Sidecar Dapr<br/>(daprd)"]
    Components["Composants externes<br/>(Redis, Kafka, Azure<br/>Service Bus, etc.)"]
    
    App -->|HTTP/gRPC<br/>localhost| Sidecar
    Sidecar -->|Gère la communication| Components
    
    style App fill:#4A90E2
    style Sidecar fill:#F5A623
    style Components fill:#7ED321
```


### Avantages du modèle sidecar

- **Agnostique au langage** : peu importe que votre service soit en C#, Go, Python ou Java, l'API est la même.
- **Découplage** : votre code ne dépend pas directement des SDK de Redis, Kafka, RabbitMQ, etc.
- **Portabilité** : passez d'un composant à l'autre (ex. : Redis → Azure Cosmos DB) en changeant simplement un fichier de configuration YAML.
- **Kubernetes-native** : Dapr s'intègre naturellement dans Kubernetes, mais fonctionne aussi en local (standalone).

# Les Building Blocks

Dapr expose ses fonctionnalités sous forme de **building blocks**, chacun résolvant un problème courant des systèmes distribués :

## 1. Invocation de service (Service Invocation)

Appeler un autre service par son nom, avec découverte automatique, load balancing, retry et mTLS intégrés.

```http
POST http://localhost:3500/v1.0/invoke/order-service/method/create
Content-Type: application/json

{
  "productId": 42,
  "quantity": 2
}
```

En .NET avec le SDK Dapr :

```csharp
var client = new DaprClientBuilder().Build();

var order = new Order { ProductId = 42, Quantity = 2 };
var result = await client.InvokeMethodAsync<Order, OrderConfirmation>(
    "order-service", "create", order);
```

## 2. Gestion d'état (State Management)

Stocker et lire de l'état clé/valeur avec des garanties de concurrence optimiste (ETags).

```csharp
// Sauvegarder un état
await client.SaveStateAsync("statestore", "order-42", order);

// Lire un état
var saved = await client.GetStateAsync<Order>("statestore", "order-42");

// Supprimer un état
await client.DeleteStateAsync("statestore", "order-42");
```

Le composant `statestore` est configuré dans un fichier YAML. On peut utiliser Redis, PostgreSQL, Azure Cosmos DB, etc., sans changer une ligne de code :

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: "localhost:6379"
```

## 3. Pub/Sub (Publish & Subscribe)

Publier et consommer des événements de manière asynchrone, avec garantie de livraison (at-least-once).

### Publier un événement

```csharp
await client.PublishEventAsync("pubsub", "orders", new OrderCreated
{
    OrderId = 42,
    CreatedAt = DateTime.UtcNow
});
```

### S'abonner à un topic

En ASP.NET Core, il suffit d'annoter un endpoint :

```csharp
app.MapPost("/orders", [Topic("pubsub", "orders")] (OrderCreated evt) =>
{
    Console.WriteLine($"Commande reçue : {evt.OrderId}");
    return Results.Ok();
});
```

Le broker sous-jacent (RabbitMQ, Kafka, Azure Service Bus, Redis Streams…) est interchangeable via la configuration :

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
```

## 4. Bindings (Input/Output Bindings)

Permet de connecter votre application à des systèmes externes (files de messages, CRON, bases de données, services cloud) en tant que source d'événements (input) ou destination (output), sans SDK spécifique.

```csharp
// Output binding : envoyer un email via SendGrid
await client.InvokeBindingAsync("sendgrid", "create", new
{
    To = "user@example.com",
    Subject = "Confirmation",
    Body = "Votre commande a été validée."
});
```

## 5. Gestion des secrets (Secrets Management)

Accéder à des secrets stockés dans un coffre-fort (Azure Key Vault, HashiCorp Vault, Kubernetes Secrets, fichier local, etc.) via une API unifiée.

```csharp
var secret = await client.GetSecretAsync("vault", "db-connection-string");
var connectionString = secret["db-connection-string"];
```

## 6. Observabilité

Dapr génère automatiquement des traces distribuées (OpenTelemetry), des métriques et des logs pour chaque appel inter-services, sans instrumentation manuelle. Il suffit de configurer un exporteur :

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprConfig
spec:
  tracing:
    samplingRate: "1"
    otel:
      endpointAddress: "http://otel-collector:4317"
      isSecure: false
      protocol: grpc
```

## 7. Acteurs (Actors)

Dapr implémente le modèle d'acteur virtuel (inspiré d'Orléans) : chaque acteur est un objet mono-thread avec état intégré, activé à la demande.

```csharp
public interface IOrderActor : IActor
{
    Task SubmitOrder(Order order);
    Task<OrderStatus> GetStatus();
}

public class OrderActor : Actor, IOrderActor
{
    public OrderActor(ActorHost host) : base(host) { }

    public async Task SubmitOrder(Order order)
    {
        await StateManager.SetStateAsync("order", order);
    }

    public async Task<OrderStatus> GetStatus()
    {
        return await StateManager.GetStateAsync<OrderStatus>("order");
    }
}
```

# Mise en place en .NET

## Installation du CLI Dapr

```bash
# Windows (PowerShell)
powershell -Command "iwr -useb https://raw.githubusercontent.com/dapr/cli/master/install/install.ps1 | iex"

# Initialisation (lance les conteneurs Redis, Zipkin, etc.)
dapr init
```

## Ajouter le SDK Dapr à un projet ASP.NET Core

```dotnetcli
dotnet add package Dapr.AspNetCore
```

## Configurer le service

```csharp
var builder = WebApplication.CreateBuilder(args);

// Ajouter les services Dapr
builder.Services.AddDaprClient();

var app = builder.Build();

// Activer le middleware Dapr (CloudEvents, Pub/Sub)
app.UseCloudEvents();
app.MapSubscribeHandler();

app.MapPost("/orders", [Topic("pubsub", "orders")] (OrderCreated evt) =>
{
    Console.WriteLine($"Commande reçue : {evt.OrderId}");
    return Results.Ok();
});

app.Run();
```

## Lancer l'application avec Dapr

```bash
dapr run --app-id my-service --app-port 5000 -- dotnet run
```

Cette commande lance le sidecar Dapr à côté de votre application, lui attribuant l'identifiant `my-service` et écoutant sur le port 5000.

# Dapr et Aspire

Pour ceux qui utilisent **.NET Aspire** (l'orchestrateur de développement local pour .NET), Dapr s'intègre facilement via le package `Aspire.Hosting.Dapr` :

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var stateStore = builder.AddDaprStateStore("statestore");
var pubSub = builder.AddDaprPubSub("pubsub");

builder.AddProject<Projects.OrderService>("order-service")
    .WithDaprSidecar()
    .WithReference(stateStore)
    .WithReference(pubSub);

builder.Build().Run();
```

Aspire gère automatiquement le lancement des sidecars Dapr et des composants associés dans l'environnement de développement.

# Composants disponibles

L'un des grands atouts de Dapr est l'écosystème de **composants** interchangeables. Voici quelques exemples :

| Building Block     | Composants disponibles                                         |
| ------------------ | -------------------------------------------------------------- |
| State Store        | Redis, PostgreSQL, Azure Cosmos DB, MongoDB, MySQL, DynamoDB…  |
| Pub/Sub            | RabbitMQ, Kafka, Azure Service Bus, Redis Streams, Pulsar…     |
| Secrets            | Azure Key Vault, HashiCorp Vault, Kubernetes Secrets, Local…   |
| Bindings           | Cron, SendGrid, Twilio, Azure Blob Storage, S3, SMTP…         |

Pour passer d'un composant à un autre, il suffit de modifier le fichier YAML de configuration, **sans toucher au code applicatif**.

# Quand utiliser Dapr ?

Dapr est particulièrement pertinent quand :

- Vous développez une **architecture microservices** et souhaitez standardiser la communication inter-services.
- Vous voulez **découpler votre code** des implémentations d'infrastructure (brokers, bases de données, secrets).
- Vous cherchez une solution **portable** entre environnements (local, Kubernetes, cloud).
- Vous voulez bénéficier de **résilience intégrée** (retry, circuit breaker, timeout) sans implémenter ces patterns manuellement.
- Vous avez des services écrits dans **différents langages** qui doivent communiquer ensemble.

Dapr n'est en revanche **pas nécessaire** pour :

- Une application monolithique simple.
- Un projet où vous n'avez qu'un seul service sans communication externe complexe.
- Des cas où vous maîtrisez parfaitement vos dépendances d'infrastructure et ne souhaitez pas de couche d'abstraction supplémentaire.

# Résumé

| Aspect                | Description                                                        |
| --------------------- | ------------------------------------------------------------------ |
| **Modèle**            | Sidecar (processus à côté de l'application)                       |
| **API**               | HTTP et gRPC sur localhost                                         |
| **Building blocks**   | Invocation, state, pub/sub, bindings, secrets, acteurs, config…    |
| **Portabilité**       | Composants interchangeables via configuration YAML                 |
| **Environnements**    | Local (standalone), Kubernetes, cloud                              |
| **Observabilité**     | Traces distribuées, métriques et logs via OpenTelemetry            |
| **Intégration .NET**  | SDK Dapr.AspNetCore + support Aspire                               |

Dapr simplifie considérablement le développement d'applications distribuées en fournissant des abstractions prêtes à l'emploi, tout en laissant la liberté de choisir et de changer les composants d'infrastructure sous-jacents.
