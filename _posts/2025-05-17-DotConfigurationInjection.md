---
title: Injection de configuration en .NET
tags: injection dotnet configuration options
---

Dans cet article, nous allons voir comment injecter proprement la configuration dans une application .NET en tirant parti du système d’injection de dépendances. À partir d’un simple fichier appsettings.json, nous verrons comment lier des sections de configuration à des classes typées, les enregistrer au démarrage de l’application, puis les consommer dans des services via le pattern Options

<!--more-->

**Principe :**

- Les valeurs de configuration sont chargées au démarrage (ex: via `IConfiguration`).
- On peut injecter `IConfiguration` ou des objets typés (pattern Options) dans les classes (contrôleurs, services, handlers, etc.).

# Exemple simple

**Soit le fichier de configuration suivant**

```json
{
  "CheminOption": {
    "MonOption": "valeur"
  }
}
```

**Et la classe correspondante de sérialisation Json**

```csharp
public class MySettings
{
    public const string Path = "CheminOption";
    public string MonOption { get; set; } = string.Empty;
}
```

**Configuration au démarrage**

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();
    // Injection de la config
    services.Configure<MySettings>(Configuration.GetSection(MySettings.Path));
}
```

> [!TIP]  
> La méthode d'extension `Configure<T>(this IServiceCollection services, IConfiguration config)` se trouve dans `Microsoft.Extensions.Options.ConfigurationExtensions`. Il faut l'ajouter via les packages NuGet.

**Injection dans un service**

```csharp
public class MyService
{
    private readonly MySettings _settings;
    public MyService(IOptions<MySettings> options)
    {
        _settings = options.Value;
    }
}
```

# Les types d'options

En .NET, il existe plusieurs variantes d’`IOptions` pour gérer la configuration :

- **IOptions<T>**  
  Fournit les valeurs de configuration typées. Usage classique : injecté dans le constructeur, accès via `.Value`.

- **IOptionsSnapshot<T>**  
  Permet d’obtenir la configuration actualisée à chaque requête (scope). Utile pour les applications web où la config peut changer entre les requêtes.

- **IOptionsMonitor<T>**  
  Permet de surveiller les changements de configuration en temps réel et d’exécuter du code quand la config change. Utile pour les applications qui doivent réagir dynamiquement aux modifications.

# Les différentes sources de configuration

En ASP.NET Core et plus généralement en .NET moderne, la configuration peut provenir de plusieurs sources, combinées dans un *configuration builder* :

- **Fichiers JSON** : `appsettings.json`, `appsettings.Development.json`, etc.
- **Variables d’environnement** : pratiques pour Docker, Kubernetes, Azure, etc.
- **Ligne de commande** : utile pour les outils en console.
- **User secrets** (en développement) : pour stocker des secrets hors du dépôt Git.
- **Providers personnalisés** : base de données, Vault, API distante…

L’ordre d’enregistrement des providers détermine la priorité : la dernière source ajoutée peut écraser les valeurs des précédentes (ex : une variable d’environnement peut surcharger `appsettings.json`).

Exemple minimal dans un `Program.cs` (ASP.NET Core) :

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Configuration
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddEnvironmentVariables()
    .AddCommandLine(args);
```

# Aller plus loin avec les options typées

L’exemple simple mappe une section plate (`CheminOption`) vers une classe. On peut de la même façon lier des objets imbriqués ou des listes.

**Exemple avec objet imbriqué et liste**

```json
{
  "MySettings": {
    "MonOption": "valeur",
    "SousOptions": {
      "UrlApi": "https://api.example.com",
      "TimeoutSeconds": 30
    },
    "Serveurs": ["server1", "server2"]
  }
}
```

```csharp
public class MySettings
{
    public const string Path = "MySettings";

    public string MonOption { get; set; } = string.Empty;
    public SousOptions SousOptions { get; set; } = new();
    public List<string> Serveurs { get; set; } = new();
}

public class SousOptions
{
    public string UrlApi { get; set; } = string.Empty;
    public int TimeoutSeconds { get; set; }
}
```

**Enregistrement dans le conteneur**

```csharp
builder.Services.Configure<MySettings>(
    builder.Configuration.GetSection(MySettings.Path));
```

On peut ensuite injecter `IOptions<MySettings>`, `IOptionsSnapshot<MySettings>` ou `IOptionsMonitor<MySettings>` selon le besoin.

# Valider la configuration

Une configuration incorrecte (clé manquante, valeur invalide) ne doit pas être découverte en production au premier appel d’un service. Il est possible de valider les options au démarrage.

## Validation via DataAnnotations

```csharp
using System.ComponentModel.DataAnnotations;

public class MyValidatedSettings
{
    public const string Path = "MyValidatedSettings";

    [Required]
    public string ConnectionString { get; set; } = string.Empty;

    [Range(1, 60)]
    public int TimeoutSeconds { get; set; }
}
```

```csharp
builder.Services
    .AddOptions<MyValidatedSettings>()
    .Bind(builder.Configuration.GetSection(MyValidatedSettings.Path))
    .ValidateDataAnnotations()
    .Validate(s => !string.IsNullOrWhiteSpace(s.ConnectionString),
              "ConnectionString ne doit pas être vide")
    .ValidateOnStart();
```

Avec `ValidateOnStart()`, l’application échouera au démarrage si la configuration est invalide, ce qui évite des erreurs plus difficiles à diagnostiquer plus tard.

# Recharger la configuration sans redémarrer

Grâce au paramètre `reloadOnChange: true` sur les fichiers JSON et à `IOptionsMonitor<T>`, il est possible de mettre à jour la configuration à chaud.

```csharp
builder.Configuration.AddJsonFile(
    "appsettings.json", optional: false, reloadOnChange: true);

builder.Services.Configure<MySettings>(
    builder.Configuration.GetSection(MySettings.Path));
```

```csharp
public class MonService
{
    private readonly IOptionsMonitor<MySettings> _optionsMonitor;

    public MonService(IOptionsMonitor<MySettings> optionsMonitor)
    {
        _optionsMonitor = optionsMonitor;

        _optionsMonitor.OnChange(settings =>
        {
            // Réagir au changement de configuration
            Console.WriteLine($"Nouvelle valeur : {settings.MonOption}");
        });
    }

    public void DoWork()
    {
        var settings = _optionsMonitor.CurrentValue;
        // Utiliser la version actuelle de la configuration
    }
}
```

# Bonnes pratiques

- Préférer **les options typées** plutôt que l’accès direct via `IConfiguration` partout.
- Centraliser les clés de configuration dans des constantes (`Path`) plutôt que de dupliquer les chaînes.
- Valider la configuration au démarrage (`ValidateOnStart`) pour échouer vite en cas d’erreur.
- Utiliser `IOptionsSnapshot<T>` dans les applications web si la configuration doit être rechargée à chaque requête.
- Utiliser `IOptionsMonitor<T>` pour des services longue durée qui doivent réagir aux changements de configuration.
- Ne pas exposer directement des valeurs sensibles (secrets) dans les logs ou les exceptions.
