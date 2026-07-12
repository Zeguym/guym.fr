---
title: "Dapr : les Bindings (input et output) en .NET"
description: "Bindings Dapr en .NET : input bindings cron et RabbitMQ, output bindings SMTP, HTTP et Azure Blob Storage, InvokeBindingAsync, sans SDK spécifique."
tags: dotnet dapr microservices bindings messaging
series: "Dapr pour les développeurs .NET"
---

Les **bindings** sont le building block de Dapr qui relie une application au monde extérieur : un déclencheur planifié, une file d'attente, un serveur SMTP, un stockage blob, une API HTTP tierce… Un **input binding** déclenche l'application quand un événement externe survient, un **output binding** lui permet d'agir sur un système externe, le tout sans embarquer le moindre SDK spécifique. On les illustre ici avec un service de traitement des factures fournisseurs, avec Dapr 1.18 et le SDK .NET 1.18.

<!--more-->
{% include series.html %}

# Le problème

Pour rappel, Dapr est un runtime qui tourne en **sidecar** à côté de chaque application : celle-ci lui parle en HTTP ou gRPC sur `localhost`, et le sidecar s'occupe de l'infrastructure, décrite dans des **composants** YAML. Les bindings sont l'une de ces API.

Prenons un service `invoice-service` chargé des factures fournisseurs. Il doit :

- **relancer les factures échues** toutes les nuits ;
- **consommer les factures** déposées par la plateforme de dématérialisation dans une file RabbitMQ ou Azure Storage Queue ;
- **envoyer un e-mail** au service comptable à chaque facture reçue ;
- **archiver le PDF** de la facture dans un stockage blob ;
- **appeler l'API HTTP** d'un service tiers qui vérifie que le fournisseur existe et est actif.

Sans Dapr, cela représente cinq SDK différents (Quartz ou un `BackgroundService`, RabbitMQ.Client, MailKit, Azure.Storage.Blobs, `HttpClient` + Polly), cinq façons de gérer la configuration, les retries et les identifiants. Les bindings remplacent tout cela par **deux gestes** : recevoir un `POST` sur un endpoint, ou appeler `InvokeBindingAsync`.

# 1. Fonctionnement

Un binding est un **composant YAML**, comme un state store ou un pub/sub (voir [la gestion d'état](/2026/03/01/dapr-etats.html) pour un autre exemple de composant). Son champ `direction` indique s'il est utilisé en entrée, en sortie, ou les deux.

```mermaid
graph LR
    Cron["Cron<br/>(planification)"]
    Queue["File d'attente<br/>(RabbitMQ, Storage Queue)"]
    Sidecar["Sidecar Dapr"]
    App["invoice-service<br/>(.NET)"]
    Smtp["SMTP"]
    Blob["Blob Storage"]
    Http["API de vérification<br/>(HTTP)"]

    Cron -->|input| Sidecar
    Queue -->|input| Sidecar
    Sidecar -->|POST /nom-du-binding<br/>localhost| App
    App -->|InvokeBindingAsync<br/>localhost| Sidecar
    Sidecar -->|output| Smtp
    Sidecar -->|output| Blob
    Sidecar -->|output| Http

    style App fill:#4A90E2
    style Sidecar fill:#F5A623
    style Cron fill:#7ED321
    style Queue fill:#7ED321
    style Smtp fill:#7ED321
    style Blob fill:#7ED321
    style Http fill:#7ED321
```

- **Input binding** : le sidecar écoute la source externe et appelle l'application en `POST` sur un endpoint qui porte, par défaut, **le nom du composant**. Répondre `200 OK` acquitte l'événement ; toute autre réponse (500…) signale un échec et l'événement est représenté. La garantie exacte (at-least-once, exactly-once) dépend du composant.
- **Output binding** : l'application appelle le sidecar en lui indiquant le nom du binding, une **opération** (`create`, `get`, `delete`, `post`…), des données et des métadonnées. Les opérations disponibles dépendent du composant.

Au démarrage, le sidecar envoie une requête `OPTIONS` sur l'endpoint de chaque input binding pour vérifier que l'application l'expose (timeout de 3 secondes par défaut, ajustable avec `--app-binding-options-timeout`).

# 2. Mise en place côté .NET

```bash
dotnet add package Dapr.AspNetCore
```

```csharp
// Program.cs : rien de spécifique aux bindings, DaprClient suffit
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDaprClient();

var app = builder.Build();
// ... endpoints d'input bindings et appels d'output bindings
app.Run();
```

Contrairement au pub/sub, il n'y a ni `UseCloudEvents()` ni `MapSubscribeHandler()` : les input bindings ne passent pas par CloudEvents, l'application reçoit le contenu brut de l'événement.

# 3. Input bindings

## Cron : une tâche planifiée sans BackgroundService

```yaml
# components/overdue-invoices.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: overdue-invoices     # => le sidecar appellera POST /overdue-invoices
spec:
  type: bindings.cron
  version: v1
  metadata:
    - name: schedule
      value: "0 0 3 * * *"   # tous les jours à 3h00 (6 champs : seconde en premier)
    - name: direction
      value: "input"
```

Le format cron de Dapr comporte **six champs** (seconde, minute, heure, jour du mois, mois, jour de la semaine). Les raccourcis `@every 15m`, `@hourly` ou `@daily` sont également acceptés, ainsi qu'un fuseau horaire via le préfixe `CRON_TZ=Europe/Paris`.

```csharp
// L'endpoint porte le nom du composant
app.MapPost("/overdue-invoices", async (ILogger<Program> logger) =>
{
    logger.LogInformation("Recherche des factures échues démarrée à {Now}", DateTime.UtcNow);

    // Logique métier : lister les factures non payées dont l'échéance est dépassée...
    await Task.CompletedTask;

    return Results.Ok(); // 200 => tick acquitté
});
```

✅ Avantage par rapport à un `BackgroundService` avec `PeriodicTimer` : la planification est externalisée dans la configuration, et changer l'horaire ne nécessite pas de redéployer le code.

❌ Attention : en Kubernetes, **chaque réplica** a son sidecar, donc chaque réplica reçoit le tick. Pour une tâche qui ne doit s'exécuter qu'une fois, il faut la rendre idempotente ou s'appuyer sur un verrou (ou sur l'API Jobs de Dapr, conçue pour ce cas).

## File d'attente : RabbitMQ

La plateforme de dématérialisation dépose chaque facture reçue dans une file `supplier-invoices`. Le même composant peut servir en entrée **et** en sortie :

```yaml
# components/supplier-invoices.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: supplier-invoices
spec:
  type: bindings.rabbitmq
  version: v1
  metadata:
    - name: queueName
      value: "supplier-invoices"
    - name: host
      value: "amqp://guest:guest@localhost:5672"
    - name: durable
      value: "true"
    - name: prefetchCount
      value: "10"              # nombre de messages en vol (input uniquement)
    - name: direction
      value: "input, output"
    - name: route
      value: "/invoices/incoming" # remplace l'endpoint par défaut /supplier-invoices
```

La métadonnée `route` permet de choisir un chemin plus parlant que le nom du composant.

```csharp
using System.Text.Json;

app.MapPost("/invoices/incoming", async (HttpRequest request, ILogger<Program> logger) =>
{
    // Le message arrive brut (pas d'enveloppe CloudEvents) : on désérialise
    // directement le corps, sans dépendre du Content-Type transmis.
    var invoice = await JsonSerializer.DeserializeAsync<SupplierInvoice>(
        request.Body, JsonSerializerOptions.Web);

    if (invoice is null)
    {
        logger.LogWarning("Message illisible, ignoré");
        return Results.Ok(); // on acquitte pour ne pas reboucler indéfiniment
    }

    try
    {
        logger.LogInformation("Facture {Number} du fournisseur {Supplier} reçue",
            invoice.InvoiceNumber, invoice.SupplierId);
        // Traitement métier (idempotent !)...
        return Results.Ok();
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Échec du traitement de {Number}", invoice.InvoiceNumber);
        return Results.StatusCode(500); // => le message sera représenté
    }
});

public record SupplierInvoice(
    string InvoiceNumber, string SupplierId, decimal Amount, DateOnly DueDate, string PdfBase64);
```

Comme avec toute file de messages, la livraison peut avoir lieu plusieurs fois : le handler doit être **idempotent** (ici, le numéro de facture fait une clé naturelle).

## Variante Azure Storage Queue

Changer de technologie de file ne touche que le YAML, le code reste identique :

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: supplier-invoices
spec:
  type: bindings.azure.storagequeues
  version: v1
  metadata:
    - name: accountName
      value: "mystorageaccount"
    - name: accountKey
      value: "***"              # à sortir dans un secret store (secretKeyRef)
    - name: queueName
      value: "supplier-invoices"
    - name: direction
      value: "input, output"
    - name: route
      value: "/invoices/incoming"
```

# 4. Output bindings

## InvokeBindingAsync : les trois surcharges

`DaprClient` expose trois surcharges :

```csharp
// 1. Fire-and-forget : données sérialisées en JSON, pas de réponse attendue
Task InvokeBindingAsync<TRequest>(string bindingName, string operation, TRequest data,
    IReadOnlyDictionary<string, string>? metadata = null, CancellationToken ct = default);

// 2. Avec réponse désérialisée
Task<TResponse> InvokeBindingAsync<TRequest, TResponse>(string bindingName, string operation,
    TRequest data, IReadOnlyDictionary<string, string>? metadata = null, CancellationToken ct = default);

// 3. Bas niveau : octets bruts + métadonnées de réponse
Task<BindingResponse> InvokeBindingAsync(BindingRequest request, CancellationToken ct = default);
```

## Envoyer un e-mail (SMTP)

```yaml
# components/email.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: email
spec:
  type: bindings.smtp
  version: v1
  metadata:
    - name: host
      value: "localhost"        # ex. smtp4dev en local
    - name: port
      value: "2525"
    - name: user
      value: "dev"
    - name: password
      value: "dev"              # en production : secretKeyRef vers un secret store
    - name: skipTLSVerify
      value: "true"             # uniquement en développement
    - name: emailFrom
      value: "factures@example.com"
    - name: emailTo
      value: "compta@example.com" # destinataire par défaut
    - name: direction
      value: "output"
```

Le binding SMTP ne supporte qu'une opération : `create`. Les champs `emailFrom`, `emailTo` et `subject` doivent être présents, soit dans le composant (valeurs par défaut), soit dans les métadonnées de la requête.

```csharp
public class AccountingNotifier(DaprClient dapr)
{
    public Task NotifyInvoiceReceivedAsync(string invoiceNumber, decimal amount, DateOnly dueDate) =>
        dapr.InvokeBindingAsync(
            bindingName: "email",
            operation: "create",
            data: $"<p>Facture {invoiceNumber} ({amount:C}) reçue, échéance le {dueDate:d}.</p>",
            metadata: new Dictionary<string, string>
            {
                // emailTo vient du composant ; on peut le surcharger ici
                // (plusieurs adresses : séparées par ';')
                ["subject"] = $"Nouvelle facture {invoiceNumber}"
            });
}
```

Pour passer à SendGrid, on remplace le composant par `bindings.twilio.sendgrid` (clé API, expéditeur par défaut) : l'appel `InvokeBindingAsync("email", "create", …)` ne change pas, seuls quelques noms de métadonnées peuvent différer d'un composant à l'autre, d'où l'intérêt de garder les valeurs par défaut dans le YAML.

## Appeler une API HTTP

```yaml
# components/supplier-check-api.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: supplier-check-api
spec:
  type: bindings.http
  version: v1
  metadata:
    - name: url
      value: "https://api.verif-entreprises.example.com"
    - name: direction
      value: "output"
```

Les opérations correspondent aux verbes HTTP (`get`, `post`, `put`, `patch`, `delete`…). La métadonnée `path` est ajoutée à l'URL de base, et toute métadonnée **commençant par une majuscule** est envoyée comme en-tête HTTP.

```csharp
public record SupplierCheckRequest(string SupplierId);
public record SupplierCheckResponse(bool IsActive, string LegalName);

public class SupplierChecker(DaprClient dapr)
{
    public async Task<SupplierCheckResponse> CheckAsync(string supplierId)
    {
        // POST https://api.verif-entreprises.example.com/v1/checks
        return await dapr.InvokeBindingAsync<SupplierCheckRequest, SupplierCheckResponse>(
            "supplier-check-api",
            "post",
            new SupplierCheckRequest(supplierId),
            new Dictionary<string, string>
            {
                ["path"] = "/v1/checks",
                ["Accept"] = "application/json" // majuscule => en-tête HTTP
            });
    }
}
```

Par défaut (`errorIfNot2XX: true`), une réponse hors 2xx lève une exception côté client.

> Pour appeler un **autre service Dapr**, on préfère l'[invocation de service](/2026/02/22/dapr-service-invocation-1.html) (découverte, mTLS). Le binding HTTP sert aux API **externes**.

## Archiver un document dans Azure Blob Storage

```yaml
# components/invoice-archive.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: invoice-archive
spec:
  type: bindings.azure.blobstorage
  version: v1
  metadata:
    - name: accountName
      value: "mystorageaccount"
    - name: accountKey
      value: "***"
    - name: containerName
      value: "invoices"
    - name: direction
      value: "output"
```

Ici on veut envoyer des octets bruts (le PDF de la facture) et récupérer l'URL du blob, renvoyée dans les métadonnées de réponse : c'est le cas d'usage de `BindingRequest`.

```csharp
public class InvoiceArchive(DaprClient dapr)
{
    public async Task<string?> ArchiveAsync(string invoiceNumber, byte[] pdf)
    {
        var request = new BindingRequest("invoice-archive", "create")
        {
            Data = pdf
        };
        request.Metadata["blobName"] = $"{invoiceNumber}.pdf"; // sinon un GUID est généré

        BindingResponse response = await dapr.InvokeBindingAsync(request);

        // La réponse de l'opération create contient l'URL du blob
        return response.Metadata.TryGetValue("blobURL", out var url) ? url : null;
    }

    public async Task<byte[]> ReadAsync(string invoiceNumber)
    {
        var request = new BindingRequest("invoice-archive", "get");
        request.Metadata["blobName"] = $"{invoiceNumber}.pdf";

        var response = await dapr.InvokeBindingAsync(request);
        return response.Data.ToArray();
    }
}
```

Le composant blob supporte aussi `delete`, `list` et `presign`. Pour du développement local, la métadonnée `endpoint` permet de pointer vers l'émulateur Azurite.

## API HTTP sous-jacente

```http
POST http://localhost:3500/v1.0/bindings/email
Content-Type: application/json

{
  "operation": "create",
  "data": "<p>Facture F-2026-0412 reçue.</p>",
  "metadata": {
    "emailTo": "compta@example.com",
    "subject": "Nouvelle facture F-2026-0412"
  }
}
```

# 5. Bindings ou Pub/Sub ?

Les deux building blocks peuvent s'appuyer sur RabbitMQ ou Kafka, d'où une confusion fréquente. La différence tient à **qui est à l'autre bout**.

| Critère | Bindings | [Pub/Sub](/2026/03/08/dapr-pub-sub.html) |
|---------|----------|---------|
| **Objectif** | Intégrer un système externe | Faire communiquer des services entre eux |
| **Modèle** | Point à point, déclencheur / action | Publication sur un topic, N abonnés |
| **Format** | Contenu brut, propre au composant | Enveloppe CloudEvents |
| **Opérations** | Spécifiques au composant (`create`, `get`, `delete`, `post`…) | `publish` / `subscribe` |
| **Abonnement** | Endpoint = nom du composant (ou `route`) | `[Topic]`, `/dapr/subscribe`, `Subscription` YAML |
| **Fonctionnalités** | Cron, SMTP, blob, HTTP, bases, files… | Routage par contenu, dead letter topics, bulk, TTL |
| **Cas typique** | Lire la file d'un prestataire, envoyer un e-mail | `InvoiceApproved` diffusé à la trésorerie et au reporting |

Règle simple : si les deux extrémités sont **nos** services Dapr, on utilise le pub/sub ; si l'une d'elles est un système **tiers** (ou un déclencheur comme cron), on utilise un binding.

# 6. Exemple complet : de la file à l'e-mail

```mermaid
sequenceDiagram
    participant Q as RabbitMQ<br/>supplier-invoices
    participant S as Sidecar Dapr
    participant A as invoice-service
    participant B as Blob Storage
    participant M as SMTP

    Q->>S: Nouveau message
    S->>A: POST /invoices/incoming
    A->>S: InvokeBinding("invoice-archive", "create")
    S->>B: Upload {invoiceNumber}.pdf
    B-->>S: blobURL
    S-->>A: BindingResponse
    A->>S: InvokeBinding("email", "create")
    S->>M: Envoi du mail
    A-->>S: 200 OK
    S->>Q: ACK
```

```csharp
using System.Text.Json;
using Dapr.Client;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDaprClient();
var app = builder.Build();

app.MapPost("/invoices/incoming", async (HttpRequest http, DaprClient dapr, ILogger<Program> logger) =>
{
    var invoice = await JsonSerializer.DeserializeAsync<SupplierInvoice>(http.Body, JsonSerializerOptions.Web);
    if (invoice is null) return Results.Ok();

    // 1. Archiver le PDF transmis dans le message
    var archive = new BindingRequest("invoice-archive", "create")
    {
        Data = Convert.FromBase64String(invoice.PdfBase64)
    };
    archive.Metadata["blobName"] = $"{invoice.InvoiceNumber}.pdf";
    var archived = await dapr.InvokeBindingAsync(archive);
    archived.Metadata.TryGetValue("blobURL", out var blobUrl);

    // 2. Prévenir la comptabilité (destinataire par défaut défini dans le composant)
    await dapr.InvokeBindingAsync("email", "create",
        $"<p>Facture {invoice.InvoiceNumber} : {invoice.Amount} EUR, échéance {invoice.DueDate:d}. PDF : {blobUrl}</p>",
        new Dictionary<string, string>
        {
            ["subject"] = $"Nouvelle facture {invoice.InvoiceNumber}"
        });

    logger.LogInformation("Facture {Number} traitée", invoice.InvoiceNumber);
    return Results.Ok(); // ACK du message RabbitMQ
});

app.MapPost("/overdue-invoices", () => Results.Ok());

app.Run();

public record SupplierInvoice(
    string InvoiceNumber, string SupplierId, decimal Amount, DateOnly DueDate, string PdfBase64);
```

Si l'envoi de l'e-mail échoue, l'exception remonte, ASP.NET Core répond 500 et le message RabbitMQ est représenté… et le PDF sera archivé une seconde fois. D'où l'importance d'opérations idempotentes (ici, `blobName` déterministe : le blob est simplement écrasé). Pour des garanties plus fortes entre écriture locale et envoi, voir le [pattern Outbox](/2026/05/23/outbox-pattern.html). Enfin, transporter le PDF en base64 dans le message reste acceptable pour quelques centaines de kilo-octets ; au-delà, on préfère faire déposer le fichier ailleurs et ne passer qu'une référence.

## Lancement en local

```bash
# Le dossier ./components contient overdue-invoices.yaml, supplier-invoices.yaml,
# email.yaml, supplier-check-api.yaml et invoice-archive.yaml
dapr run --app-id invoice-service --app-port 5000 --dapr-http-port 3500 \
    --resources-path ./components -- dotnet run --project InvoiceService

# Tester l'output binding SMTP directement via l'API HTTP du sidecar
# (port fixé avec --dapr-http-port 3500 au lancement)
curl -X POST http://localhost:3500/v1.0/bindings/email \
  -H "Content-Type: application/json" \
  -d '{"operation":"create","data":"Test","metadata":{"emailTo":"moi@example.com","subject":"Test"}}'
```

# 7. Scopes et résilience

Comme les autres composants, un binding peut être restreint à certains services avec `scopes`, et protégé par une politique de résilience (voir [l'article sur Polly](/2026/02/15/polly.html) pour les concepts de retry et circuit breaker) :

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: resiliency
spec:
  policies:
    retries:
      smtpRetry:
        policy: exponential
        maxInterval: 30s
        maxRetries: 5
  targets:
    components:
      email:
        outbound:              # appels InvokeBindingAsync
          retry: smtpRetry
      supplier-invoices:
        inbound:               # livraison à l'application
          retry: smtpRetry
```

# Bonnes pratiques

| Pratique | Pourquoi |
|----------|----------|
| **Handlers idempotents** | Les input bindings peuvent livrer plusieurs fois un même événement |
| **Répondre 200 seulement après traitement** | Une réponse prématurée acquitte un message qui pourrait être perdu |
| **Valeurs par défaut dans le YAML** | `emailFrom`, `emailTo`, `containerName`… : le code reste portable entre composants |
| **`route` explicite** | Découple le nom du composant du chemin de l'API |
| **Pas de secret en clair** | `password`, `accountKey` : `secretKeyRef` vers un secret store Dapr |
| **Scopes** | Seuls les services concernés doivent pouvoir utiliser un binding |
| **Cron + plusieurs réplicas** | Prévoir l'exécution multiple ou utiliser l'API Jobs |

# Résumé

| Aspect | Détail |
|--------|--------|
| **Input binding** | Le sidecar appelle `POST /{nom-du-composant}` (ou `route`) ; 200 = ACK, autre = retry |
| **Output binding** | `POST http://localhost:3500/v1.0/bindings/{nom}` avec `operation`, `data`, `metadata` |
| **SDK .NET** | `DaprClient.InvokeBindingAsync` (3 surcharges), `BindingRequest` / `BindingResponse` |
| **Direction** | Métadonnée `direction` : `input`, `output` ou `input, output` |
| **Composants courants** | `bindings.cron`, `bindings.rabbitmq`, `bindings.azure.storagequeues`, `bindings.smtp`, `bindings.twilio.sendgrid`, `bindings.http`, `bindings.azure.blobstorage` |
| **Différence avec Pub/Sub** | Système externe et contenu brut vs services internes et CloudEvents |

Les bindings sont souvent le building block le plus sous-estimé de Dapr : ils font disparaître une bonne partie du code d'intégration (planificateurs, clients de files, clients SMTP ou blob) au profit d'un endpoint et d'un appel `InvokeBindingAsync`. Un seul point reste à traiter avant la production : les mots de passe et clés d'accès écrits en clair dans les composants de cet article doivent être remplacés par des références `secretKeyRef` vers un secret store, que Dapr résout lui-même au chargement du composant.
