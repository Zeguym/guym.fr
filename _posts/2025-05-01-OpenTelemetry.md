---
title: Open Telemery
tags: dotnet opentelemetry
---

# Qu'est ce qu'opentelemetry

OpenTelemetry est un standard open source pour la collecte, l’export et l’analyse des données de télémétrie : **traces**, **métriques** et **logs**. Il permet d’observer le comportement de vos applications (performances, erreurs, dépendances) de façon unifiée, quel que soit le langage ou la plateforme.

<!--more-->

## À quoi ça sert ?

- **Observabilité** : comprendre le fonctionnement interne des applications distribuées.
- **Traces** : suivre le parcours d’une requête à travers plusieurs services (distributed tracing).
- **Métriques** : surveiller les performances (temps de réponse, nombre de requêtes, etc.).
- **Logs** : centraliser et corréler les messages de log avec les traces.

## Fonctionnement

1. **Instrumentation** : Ajout de bibliothèques OpenTelemetry dans votre code ou utilisation d’auto-instrumentation.
2. **Collecte** : Les données sont collectées automatiquement ou via des API.
3. **Export** : Les données sont envoyées vers des outils d’observabilité (ex : Jaeger, Prometheus, Azure Monitor, Grafana, etc.).

## Avantages

- **Standardisé** : même API pour tous les langages.
- **Interopérable** : compatible avec de nombreux outils d’analyse.
- **Extensible** : ajoutez vos propres métriques ou traces personnalisées.

# En dotnet

Pour ajouter OpenTelemetry dans une application ASP.NET Core, il suffit d’installer les packages nécessaires et de configurer les services dans `Program.cs` ou `Startup.cs`.

Voici un exemple simple pour activer la collecte des traces et exporter vers la console :

## Collecte

Installer les packages NuGet suivants :

```dotnetcli
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
```

### Les traces

```csharp
// Ajout d'OpenTelemetry pour les traces
builder.Services.AddOpenTelemetry()
    .WithTracing(tracerProviderBuilder =>
    {
        tracerProviderBuilder
            .AddAspNetCoreInstrumentation() //Instrumentation automatique des requêtes HTTP
    });
```

### Les métriques

```csharp
// Ajout d'OpenTelemetry pour les traces
builder.Services.AddOpenTelemetry()
    .WithMetrics(metricsProviderBuilder =>
    {
        metricsProviderBuilder
            .AddAspNetCoreInstrumentation() // métriques HTTP automatiques
    });
```

### Les métriques

```csharp
// Ajout d'OpenTelemetry pour les traces
builder.Logging.AddOpenTelemetry(options =>
{
    options.IncludeFormattedMessage = true; // inclut le message formaté
    options.ParseStateValues = true;        // parse les valeurs d'état
});
```

# Open Telemetry Protocol (OTLP)

OTLP (**OpenTelemetry Protocol**) est le protocole standardisé utilisé par OpenTelemetry pour transmettre les données de télémétrie (**traces**, **métriques**, **logs**) entre les applications instrumentées et les outils d’observabilité (comme Jaeger, Prometheus, Grafana, etc.).

## Caractéristiques principales

- **Standardisé** : conçu pour être universel, quel que soit le langage ou la plateforme.
- **Supporte plusieurs formats** : gRPC (par défaut, performant) et HTTP/Protobuf.
- **Unifie** l’envoi des traces, métriques et logs via un seul protocole.

## Utilisation

- Les SDK OpenTelemetry exportent les données via OTLP vers des backends compatibles (Jaeger, Tempo, Azure Monitor, etc.).
- Il suffit de configurer un exporteur OTLP dans l'application.

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracerProviderBuilder =>
    {
        tracerProviderBuilder
            .AddAspNetCoreInstrumentation() //Instrumentation automatique des requêtes HTTP
            .AddOtlpExporter();
    });
```

La méthode `.AddOtlpExporter()` dans la configuration OpenTelemetry permet d’exporter les données de télémétrie (**traces**, **métriques**, ou **logs**) via le protocole OTLP (OpenTelemetry Protocol) vers un backend compatible (comme Jaeger, Tempo, Azure Monitor, etc.).

- Elle ajoute un exporteur OTLP à la pipeline OpenTelemetry.
- Par défaut, elle envoie les données à l’endpoint OTLP standard (`http://localhost:4317` en gRPC).
- Vous pouvez personnaliser l’endpoint, le protocole (gRPC ou HTTP/Protobuf) et d’autres options via une surcharge :

```csharp
.AddOtlpExporter(options =>
{
    options.Endpoint = new Uri("http://localhost:4317");
    options.Protocol = OpenTelemetry.Exporter.OtlpExportProtocol.Grpc;
});
```

Ainsi, il est possible d’adapter la configuration de l’exporteur OTLP selon l’infrastructure ou les besoins spécifiques.

En .NET avec OpenTelemetry il ya deux méthodes de configuration OTLP:

- **`AddOtlpExporter`**  
  Cette méthode est utilisée dans la configuration des services (ex : `builder.Services.AddOpenTelemetry()`) pour ajouter un exporteur OTLP à la pipeline de télémétrie. Elle est utilisée lors de la configuration du Tracing, des Metrics ou des Logs dans le code C#. Il faut l'appeler sur chacun des points de collectes (traces, métriques, logs)

  Exemple :

  ```csharp
  builder.Services.AddOpenTelemetry()
      .WithTracing(tracerProviderBuilder =>
      {
          tracerProviderBuilder.AddOtlpExporter();
      });
  ```

- **`UseOtlpExporter`**  
  Cette méthode est généralement utilisée dans la configuration via le fichier `appsettings.json` ou dans certains scénarios avancés, notamment avec les extensions de configuration OpenTelemetry. Elle permet d’activer l’exporteur OTLP sans modifier le code, mais en passant par la configuration.
  ```csharp
  builder.Services.AddOpenTelemetry().UseOtlpExporter(à)
      .WithTracing(tracerProviderBuilder => { })
      .WithMetrics(mertricsProviderBuilder => { });
  ```
  toute les infos sur [son utilisation est ici](https://github.com/open-telemetry/opentelemetry-dotnet/blob/fb74013d644d56f44ef89dd57358947bd9f68345/src/OpenTelemetry.Exporter.OpenTelemetryProtocol/README.md#enable-otlp-exporter-for-all-signals)

# Collecte

Le plus simple est d'utilser le dashborad d'aspire via docker

```powershell
docker run --rm -it -p 18888:18888 -p 4317:18889 -e DOTNET_DASHBOARD_UNSECURED_ALLOW_ANONYMOUS=true --name aspire-dashboard mcr.microsoft.com/dotnet/aspire-dashboard:latest
```

<img alt="dash bord aspire" src="https://guym.fr/assets/images/aspire-dashboard-trace.png" />

## En production

L’ajout de la télémétrie via OpenTelemetry dans une application .NET (comme ton projet Razor Pages) peut impacter les performances en production, principalement pour les raisons suivantes :

1. **Surcharge CPU et mémoire**
   La collecte de traces, métriques et logs ajoute du travail supplémentaire à chaque requête ou opération. Si le volume de données collectées est élevé (beaucoup de spans, logs détaillés, métriques fréquentes), cela peut augmenter l’utilisation CPU et mémoire.

2. **Latence réseau**
   OpenTelemetry exporte les données vers des backends (ex : Jaeger, Prometheus, OTLP collector) via le réseau. Si l’export est synchrone ou mal configuré, cela peut ralentir les réponses HTTP ou les traitements.

3. **Blocages ou contention**
   Certains instrumentations (ex : Entity Framework, HTTP client) peuvent ajouter des hooks sur chaque appel, ce qui peut ralentir les accès à la base ou les appels externes. Si le buffer d’export est saturé, des blocages ou des pertes de données peuvent survenir.

4. **Volume de logs et traces**
   Un niveau de détail trop élevé (ex : logs de debug, traces sur toutes les méthodes) peut générer beaucoup de données, ce qui surcharge le système et le backend de télémétrie.

5. **Configuration non optimale**
   Par défaut, certains instrumentations collectent beaucoup d’informations. Il est important d’ajuster la configuration (sampling, filtres, enrichissement) pour limiter l’impact.

**Bonnes pratiques pour limiter l’impact**

- **Utiliser le sampling** : ne pas tout tracer, mais seulement un pourcentage des requêtes.
- **Exporter en mode batch/asynchrone** : éviter l’export synchrone qui bloque les threads.
- **Limiter les enrichissements et tags** : n’ajouter que les informations utiles.
- **Surveiller les ressources** : monitorer l’impact de la télémétrie sur le CPU, la mémoire et la latence.
- **Adapter la configuration selon l’environnement** : plus de détails en dev, moins en prod.
