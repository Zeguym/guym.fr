---
title: Inversion de contrôle et injection de dépendance pour une assembly (Ioc 2/3)
tags: dotnet ioc dll injection-dependance
series: "IoC et DI en .NET"
---

Lorsqu'une bibliothèque de classes (DLL) est créée, il est important de permettre à cette bibliothèque de s'intégrer facilement avec le conteneur IoC de l'application principale qui l'utilise. Cela permet de gérer les dépendances de manière efficace et de favoriser la réutilisabilité du code.

<!--more-->
{% include series.html %}
## L'injection de dépendances dans une assembly (DLL) en .NET

Pour configurer l'injection de dépendances dans une **DLL** (bibliothèque de classes) en .NET, vous devez permettre à la bibliothèque de s'intégrer avec le conteneur IoC de l'application principale. Voici les étapes pour y parvenir :

### Ajouter une méthode d'extension pour l'enregistrement des services

Dans votre DLL, créez une méthode d'extension qui enregistre les services spécifiques à la bibliothèque dans le conteneur IoC.

**Exemple :**

```csharp
public static class DependencyInjectionExtensions
{
    public static IServiceCollection AddMyLibraryServices(this IServiceCollection services)
    {
        // Enregistrer les services spécifiques à la DLL
        services.AddTransient<IMyService, MyService>();
        services.AddScoped<IMyRepository, MyRepository>();

        return services;
    }
}
```

### Appeler la méthode d'extension dans l'application principale

Dans l'application principale (ex. : projet ASP.NET Core), appelez cette méthode pour enregistrer les services de la DLL dans le conteneur IoC.

**Exemple :**

```csharp
var builder = WebApplication.CreateBuilder(args);

// Enregistrement des services de la DLL
builder.Services.AddMyLibraryServices();

var app = builder.Build();

app.Run();
```

### Structurer les dépendances dans la DLL

Assurez-vous que les classes de votre DLL respectent les principes de l'injection de dépendance :

- Utilisez des interfaces pour définir les contrats.
- Injectez les dépendances via le constructeur.
- Les classes concrètes doivent être internes ou publiques selon le besoin.

**Exemple :**

```csharp
public interface IMyService
{
    void DoWork();
}

internal class MyService : IMyService
{
    private readonly IMyRepository _repository;

    public MyService(IMyRepository repository)
    {
        _repository = repository;
    }

    public void DoWork()
    {
        _repository.SaveData("Some data");
    }
}
```

### 4. Ajouter des dépendances externes (si nécessaire)

Si votre DLL dépend d'autres bibliothèques ou services (ex. : `DbContext`, configuration, etc.), vous pouvez les injecter via le conteneur IoC.

**Exemple** avec `DbContext` :

```csharp
public static class DependencyInjectionExtensions
{
    public static IServiceCollection AddMyLibraryServices(this IServiceCollection services, string connectionString)
    {
        services.AddDbContext<MyDbContext>(options =>
            options.UseSqlServer(connectionString));

        services.AddTransient<IMyService, MyService>();
        services.AddScoped<IMyRepository, MyRepository>();

        return services;
    }
}
```

Dans l'application principale :

```csharp
var builder = WebApplication.CreateBuilder(args);

string connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
builder.Services.AddMyLibraryServices(connectionString);

var app = builder.Build();

app.Run();
```

### 5. Tests unitaires pour la DLL

Pour tester les services de votre DLL, vous pouvez utiliser un conteneur IoC simulé ou des mocks.

**Exemple** avec Moq :

```csharp
var mockRepository = new Mock<IMyRepository>();
mockRepository.Setup(r => r.SaveData(It.IsAny<string>()));

var service = new MyService(mockRepository.Object);
service.DoWork();

mockRepository.Verify(r => r.SaveData(It.IsAny<string>()), Times.Once);
```

### 6. Bonnes pratiques

- **Isolation** : La DLL ne doit pas dépendre de l'application principale. Elle doit être autonome et réutilisable.
- **Configuration flexible** : Si la DLL nécessite des configurations spécifiques, exposez-les via des paramètres dans la méthode d'extension.
- **Documentation** : Documentez la méthode d'extension pour expliquer comment intégrer la DLL dans une application.

## Ajouter des paramètres optionnels à la méthode d'extension de configuration des services

Pour ajouter un **paramètre optionnel** à la méthode `AddMyLibraryServices`, vous pouvez utiliser un objet de configuration ou un paramètre par défaut. Cela permet de personnaliser le comportement de la méthode tout en gardant une configuration flexible.

### La classe d'options

1. **Créer une classe d'options** :

```csharp
public class MyLibraryOptions
{
    public bool EnableLogging { get; set; }
    public string CustomSetting { get; set; }
}
```

2. **Modifier la méthode d'extension** :

```csharp
public static class DependencyInjectionExtensions
{
    public static IServiceCollection AddMyLibraryServices(this IServiceCollection services, Action<MyLibraryOptions> configureOptions)
    {
        // Configurer les options
        var options = new MyLibraryOptions();
        configureOptions(options);

        // Enregistrer les services spécifiques à la DLL
        services.AddTransient<IMyService, MyService>();
        services.AddScoped<IMyRepository, MyRepository>();

        // Ajouter un service conditionnel basé sur les options
        if (options.EnableLogging)
        {
            services.AddSingleton<ILogger, ConsoleLogger>();
        }

        return services;
    }
}
```

3. **Appeler la méthode dans l'application principale** :

```csharp
builder.Services.AddMyLibraryServices(options =>
{
    options.EnableLogging = true;
    options.CustomSetting = "CustomValue";
});
```

### Utiliser des options liées à la configuration

Si vous souhaitez lier les options à une section de configuration (ex. : `appsettings.json`), utilisez le modèle `IOptions<T>`.

1. **Modifier la méthode d'extension** :

```csharp
public static class DependencyInjectionExtensions
{
    public static IServiceCollection AddMyLibraryServices(this IServiceCollection services, IConfiguration configuration)
    {
        // Lier les options à une section de configuration
        services.Configure<MyLibraryOptions>(configuration.GetSection("MyLibrary"));

        // Enregistrer les services spécifiques à la DLL
        services.AddTransient<IMyService, MyService>();
        services.AddScoped<IMyRepository, MyRepository>();

        // Ajouter un service conditionnel basé sur les options
        var options = configuration.GetSection("MyLibrary").Get<MyLibraryOptions>();
        if (options.EnableLogging)
        {
            services.AddSingleton<ILogger, ConsoleLogger>();
        }

        return services;
    }
}
```

2. **Ajouter une section dans `appsettings.json`** :

```json
"MyLibrary": {
  "EnableLogging": true,
  "CustomSetting": "CustomValue"
}
```

3. **Appeler la méthode dans l'application principale** :

```csharp
builder.Services.AddMyLibraryServices(builder.Configuration);
```

Avec ces approches, vous pouvez facilement ajouter des paramètres optionnels à votre méthode d'extension pour configurer les services de votre DLL, tout en offrant une flexibilité maximale aux utilisateurs de la bibliothèque.
