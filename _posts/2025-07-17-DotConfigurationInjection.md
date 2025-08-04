---
title: Injection de configuration en .NET
tags: injection dotnet
---

L’injection de configuration en .NET permet d’accéder aux paramètres (appsettings.json, variables d’environnement, etc.) via le système d’injection de dépendances.

**Principe :**
- Les valeurs de configuration sont chargées au démarrage (ex: via IConfiguration).
- On peut injecter IConfiguration ou des objets typés (Options pattern) dans les classes (contrôleurs, services).

# Exemple simple

**Soit le fichier de configuration suivant**

````json
{
    "CheminOption"
    {
        "MonOption" :"valeur"
    }
 }
````

**Et la classe correspondante de sérialisation Json**

````csharp
public class MySettings
{
    static readonly string Path = "CheminOption"
    string MonOption {get;set;}
}
````

**Configuration au démarrage**

````csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();
    // Injection de la config
    services.Configure<MySettings>(Configuration.GetSection(MySettings.Path));
}
````

> [!TIP]  
> La méthode d'extention `Configure<T>(this IServiceCollection services, IConfiguration config)` se trouve dans Microsoft.Extensions.DependencyInjection. IL faut l'ajouter via les Nugets.


**Injection dans un service**

````csharp
public class MyService
{
    private readonly MySettings _settings;
    public MyService(IOptions<MySettings> options)
    {
        _settings = options.Value;
    }
}
````

# Les types d'options

En .NET, il existe plusieurs variantes d’`IOptions` pour gérer la configuration :

- **IOptions<T>**  
  Fournit les valeurs de configuration typées. Usage classique : injecté dans le constructeur, accès via `.Value`.

- **IOptionsSnapshot<T>**  
  Permet d’obtenir la configuration actualisée à chaque requête (scope). Utile pour les applications web où la config peut changer entre les requêtes.

- **IOptionsMonitor<T>**  
  Permet de surveiller les changements de configuration en temps réel et d’exécuter du code quand la config change. Utile pour les applications qui doivent réagir dynamiquement aux modifications.

