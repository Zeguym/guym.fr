---
title: Mesurer et diagnostiquer les performances EF Core
tags: dotnet efcore performance observability logging opentelemetry
series: "Entity Framework"
---

Entity Framework Core : instrumenter les requêtes, lire les métriques et corriger les vrais goulots d'étranglement

<!--more-->
{% include series.html %}

# 1. Pourquoi mesurer

Après les optimisations techniques (requêtes précompilées, cache, DbContext pooling), il reste une règle essentielle :

- on ne corrige pas une impression,
- on corrige une mesure.

Sans observabilité, on peut passer des heures à optimiser la mauvaise zone.

# 2. Activer un niveau de logs utile

La première étape consiste à rendre les requêtes visibles.

## 2.1. Logging EF Core

```csharp
builder.Services.AddDbContextPool<AppDbContext>(options =>
{
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"));

    options.LogTo(Console.WriteLine, LogLevel.Information)
           .EnableDetailedErrors();
});
```

En développement, ce niveau de log permet de voir :

- la requête SQL envoyée,
- la durée d'exécution,
- les paramètres (si activés explicitement).

## 2.2. Attention aux données sensibles

`EnableSensitiveDataLogging()` est pratique en debug mais à éviter en production.

```csharp
if (builder.Environment.IsDevelopment())
{
    options.EnableSensitiveDataLogging();
}
```

# 3. Taguer les requêtes pour les retrouver

Quand plusieurs endpoints exécutent des requêtes proches, les tags facilitent l'analyse.

```csharp
var products = await _db.Products
    .TagWith("ProductsController.GetByCategory")
    .AsNoTracking()
    .Where(p => p.CategoryId == categoryId)
    .OrderBy(p => p.Name)
    .Select(p => new ProductDto(p.Id, p.Name, p.Price))
    .ToListAsync(ct);
```

Avec `TagWith`, on remonte plus vite la requête dans les logs, le profiler SQL ou l'APM.

## 3.1. Correlation ID et trace distribuée

Dans un système distribué, l'objectif n'est pas seulement d'identifier la requête SQL, mais de la rattacher à une trace de bout en bout.

Exemple : injecter un `correlation id` métier et le `trace id` OpenTelemetry dans le tag.

```csharp
using System.Diagnostics;

var correlationId = httpContext.TraceIdentifier;
var traceId = Activity.Current?.TraceId.ToString() ?? "no-trace";

var products = await _db.Products
    .TagWith($"op=Products.GetByCategory;corr={correlationId};trace={traceId}")
    .AsNoTracking()
    .Where(p => p.CategoryId == categoryId)
    .Select(p => new ProductDto(p.Id, p.Name, p.Price))
    .ToListAsync(ct);
```

Résultat : dans les logs SQL, dans l'APM et dans vos traces, vous avez la même clé de corrélation.

## 3.2. TagWithCallSite pour retrouver l'origine du code

Quand plusieurs requêtes se ressemblent, `TagWithCallSite()` est pratique pour savoir rapidement d'où vient l'appel (fichier/méthode/ligne selon le contexte d'exécution).

```csharp
var products = await _db.Products
    .TagWithCallSite("Products query")
    .AsNoTracking()
    .Where(p => p.CategoryId == categoryId)
    .Select(p => new ProductDto(p.Id, p.Name, p.Price))
    .ToListAsync(ct);
```

`TagWithCallSite()` est très utile en investigation, mais il ne remplace pas un tag métier stable ni un correlation id.

## 3.3. Auto-tagging pour éviter le manuel partout

Faire `TagWith(...)` à la main dans chaque requête est vite pénible. Une approche robuste consiste à ajouter un `DbCommandInterceptor` qui préfixe automatiquement le SQL avec les identifiants de corrélation.

```csharp
using System.Data.Common;
using System.Diagnostics;
using Microsoft.EntityFrameworkCore.Diagnostics;

public sealed class CorrelationCommandInterceptor : DbCommandInterceptor
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CorrelationCommandInterceptor(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    private void AddComment(DbCommand command)
    {
        var corr = _httpContextAccessor.HttpContext?.TraceIdentifier ?? "no-corr";
        var trace = Activity.Current?.TraceId.ToString() ?? "no-trace";

        command.CommandText = $"-- corr:{corr}; trace:{trace} \n{command.CommandText}";
    }

    public override InterceptionResult<DbDataReader> ReaderExecuting(
        DbCommand command,
        CommandEventData eventData,
        InterceptionResult<DbDataReader> result)
    {
        AddComment(command);
        return base.ReaderExecuting(command, eventData, result);
    }

    public override InterceptionResult<object> ScalarExecuting(
        DbCommand command,
        CommandEventData eventData,
        InterceptionResult<object> result)
    {
        AddComment(command);
        return base.ScalarExecuting(command, eventData, result);
    }

    public override InterceptionResult<int> NonQueryExecuting(
        DbCommand command,
        CommandEventData eventData,
        InterceptionResult<int> result)
    {
        AddComment(command);
        return base.NonQueryExecuting(command, eventData, result);
    }
}
```

Enregistrement :

```csharp
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<CorrelationCommandInterceptor>();

builder.Services.AddDbContextPool<AppDbContext>((sp, options) =>
{
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"));
    options.AddInterceptors(sp.GetRequiredService<CorrelationCommandInterceptor>());
});
```

Bonnes pratiques :

- garder des tags courts et stables,
- ne jamais inclure de données sensibles,
- standardiser le format des clés (`op`, `corr`, `trace`) pour faciliter les recherches.

# 4. Mesurer avec OpenTelemetry

## 4.1. Instrumentation minimale

```csharp
builder.Services
    .AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddSqlClientInstrumentation();
    });
```

Cette configuration donne une vue bout en bout :

- requête HTTP entrante,
- appels SQL réalisés,
- durée totale et durée par dépendance.

## 4.2. Ce qu'il faut regarder en priorité

- p95 et p99 de latence (pas seulement la moyenne),
- nombre de requêtes SQL par endpoint,
- requêtes les plus lentes,
- fréquence des timeouts.


