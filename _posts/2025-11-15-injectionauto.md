---
title: Injection automatique de dépendances en .NET (Ioc 3/3)
tags: dotnet ioc source-generator injection-dependance
series: "IoC et DI en .NET"
---

L'injection de dépendances (IoC) est une pratique courante en développement .NET pour gérer les dépendances entre les classes. Lorsqu'une application devient plus grande, l'enregistrement manuel de chaque service dans le conteneur IoC peut devenir fastidieux et sujet à erreurs.

L'enregistrement automatique des services permet de simplifier ce processus en détectant et en enregistrant les types d'une assembly (DLL) en fonction de leurs interfaces. S"'il existe de nombreuses bibliothèques tierces qui offrent cette fonctionnalité, il est également possible de l'implémenter soi-même simplement. Voici deux méthodes courantes : une utilisant la réflexion et une autre utilisant un générateur de code (Source Generator).

<!--more-->
{% include series.html %}
## Méthode classique par réflexion

### 1. Exemple de méthode d'enregistrement automatique

Créez une méthode d'extension qui parcourt tous les types d'une assembly et enregistre les implémentations correspondant à leurs interfaces.

```csharp
public static class DependencyInjectionExtensions
{
    public static IServiceCollection AddServicesFromAssembly(this IServiceCollection services, Assembly assembly)
    {
        // Parcourir tous les types de l'assembly
        var types = assembly.GetTypes()
            .Where(t => t.IsClass && !t.IsAbstract) // Filtrer uniquement les classes concrètes
            .ToList();

        foreach (var type in types)
        {
            // Trouver les interfaces implémentées par la classe
            var interfaces = type.GetInterfaces();
            foreach (var @interface in interfaces)
            {
                // Enregistrer la classe avec son interface
                services.AddTransient(@interface, type);
            }
        }

        return services;
    }
}
```

### 2. Utilisation dans l'application principale

Dans l'application principale, utilisez cette méthode pour enregistrer automatiquement tous les services d'une assembly.

 **Exemple :**

```csharp
var builder = WebApplication.CreateBuilder(args);

// Charger l'assembly contenant les services
var assembly = Assembly.Load("MyLibrary"); // Remplacez "MyLibrary" par le nom de votre DLL

// Enregistrer automatiquement les services de l'assembly
builder.Services.AddServicesFromAssembly(assembly);

var app = builder.Build();

app.Run();
```

### 3. Exemple avec une classe et une interface dans la DLL

#### Interface et implémentation dans la DLL :

```csharp
public interface IMyService
{
    void DoWork();
}

public class MyService : IMyService
{
    public void DoWork()
    {
        Console.WriteLine("Work done!");
    }
}
```

#### Résultat :

La méthode `AddServicesFromAssembly` détectera automatiquement que `MyService` implémente `IMyService` et enregistrera cette relation dans le conteneur IoC.

### 4. Enregistrement conditionnel

Si vous souhaitez enregistrer uniquement certains types (par exemple, ceux qui respectent une convention de nommage), vous pouvez ajouter des filtres.

**Exemple** avec un filtre :

```csharp
var types = assembly.GetTypes()
    .Where(t => t.IsClass && !t.IsAbstract && t.Name.EndsWith("Service")) // Filtrer par convention
    .ToList();
```

### 5. Cycle de vie des services

Par défaut, l'exemple ci-dessus utilise `AddTransient`. Vous pouvez personnaliser le cycle de vie (Transient, Scoped, Singleton) en fonction de vos besoins.

**Exemple** avec un cycle de vie configurable :

```csharp
public static IServiceCollection AddServicesFromAssembly(this IServiceCollection services, Assembly assembly, ServiceLifetime lifetime = ServiceLifetime.Transient)
{
    var types = assembly.GetTypes()
        .Where(t => t.IsClass && !t.IsAbstract)
        .ToList();

    foreach (var type in types)
    {
        var interfaces = type.GetInterfaces();
        foreach (var @interface in interfaces)
        {
            switch (lifetime)
            {
                case ServiceLifetime.Singleton:
                    services.AddSingleton(@interface, type);
                    break;
                case ServiceLifetime.Scoped:
                    services.AddScoped(@interface, type);
                    break;
                default:
                    services.AddTransient(@interface, type);
                    break;
            }
        }
    }

    return services;
}
```

### 6. Points importants

- **Performance** : L'utilisation de la réflexion peut avoir un impact sur les performances au démarrage. Assurez-vous que cela est acceptable pour votre application.
- **Convention** : Si vous avez des classes qui implémentent plusieurs interfaces, vérifiez que cela ne crée pas de conflits.
- **Assemblies multiples** : Si vous devez enregistrer des services provenant de plusieurs assemblies, vous pouvez appeler la méthode pour chaque assembly ou les charger dynamiquement.

### 7. Exemple avec plusieurs assemblies

Si vous avez plusieurs DLLs, vous pouvez les charger dynamiquement :

```csharp
var assemblies = AppDomain.CurrentDomain.GetAssemblies()
    .Where(a => a.FullName.StartsWith("MyNamespace"))
    .ToList();

foreach (var assembly in assemblies)
{
    builder.Services.AddServicesFromAssembly(assembly);
}
```

Cette approche permet de simplifier l'enregistrement des services dans des projets modulaires ou des bibliothèques, tout en rendant le code plus maintenable.

## Méthode avec un Source Generator

Une autre approche plus avancée consiste à utiliser un **Source Generator** pour générer automatiquement le code d'enregistrement des services à la compilation. Cela permet d'éviter l'overhead de la réflexion au runtime.

Pour automatiser l'enregistrement des services d'une assembly en utilisant un **code generator**, vous pouvez tirer parti de **Source Generators** introduits dans .NET 5. Cela permet de générer du code au moment de la compilation pour enregistrer automatiquement les services dans le conteneur IoC.

### 1. Pourquoi utiliser un code generator ?

- Évite l'utilisation de la réflexion, ce qui améliore les performances au démarrage.
- Génère du code statique, ce qui facilite le débogage et la maintenance.
- Permet de personnaliser l'enregistrement des services en fonction de conventions.

### 2. Étapes pour créer un code generator

#### a) Créer un projet Source Generator

1. Ajoutez un nouveau projet de type **Class Library** à votre solution.
2. Modifiez le fichier `.csproj` pour inclure les métadonnées nécessaires :
   ```xml
   <Project Sdk="Microsoft.NET.Sdk">
     <PropertyGroup>
       <OutputType>Library</OutputType>
       <TargetFramework>netstandard2.0</TargetFramework>
       <LangVersion>latest</LangVersion>
       <IsRoslynAnalyzer>true</IsRoslynAnalyzer>
     </PropertyGroup>
     <ItemGroup>
       <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.0.1" PrivateAssets="all" />
     </ItemGroup>
   </Project>
   ```

#### b) Implémenter le Source Generator

Créez une classe qui hérite de `IIncrementalGenerator`. Voici un exemple :

```csharp
using System.Text;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Text;

[Generator]
public class ServiceRegistrationGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // Ajouter un pipeline pour analyser les fichiers source
        var classesWithInterfaces = context.SyntaxProvider
            .CreateSyntaxProvider(
                predicate: static (node, _) => IsClassWithInterfaces(node),
                transform: static (context, _) => GetClassWithInterfaces(context))
            .Where(static classInfo => classInfo != null);

        // Combiner les résultats et générer le code
        context.RegisterSourceOutput(classesWithInterfaces.Collect(), (spc, classes) =>
        {
            GenerateServiceRegistrationCode(spc, classes!);
        });
    }

    private static bool IsClassWithInterfaces(SyntaxNode node)
    {
        // Vérifie si le nœud est une déclaration de classe avec des interfaces
        return node is ClassDeclarationSyntax classDeclaration &&
               classDeclaration.BaseList != null &&
               classDeclaration.BaseList.Types.Any();
    }

    private static (string Interface, string Implementation)? GetClassWithInterfaces(GeneratorSyntaxContext context)
    {
        var classDeclaration = (ClassDeclarationSyntax)context.Node;
        var classSymbol = context.SemanticModel.GetDeclaredSymbol(classDeclaration) as INamedTypeSymbol;

        if (classSymbol == null)
            return null;

        // Récupérer les interfaces implémentées par la classe
        foreach (var @interface in classSymbol.Interfaces)
        {
            return (@interface.ToDisplayString(), classSymbol.ToDisplayString());
        }

        return null;
    }

    private static void GenerateServiceRegistrationCode(SourceProductionContext context, ImmutableArray<(string Interface, string Implementation)> classes)
    {
        var sourceBuilder = new StringBuilder(@"
using Microsoft.Extensions.DependencyInjection;

namespace Generated
{
    public static class ServiceRegistration
    {
        public static IServiceCollection AddGeneratedServices(this IServiceCollection services)
        {
");

        foreach (var (interfaceName, implementationName) in classes.Distinct())
        {
            sourceBuilder.AppendLine($@"            services.AddTransient<{interfaceName}, {implementationName}>();");
        }

        sourceBuilder.AppendLine(@"
            return services;
        }
    }
}");

        context.AddSource("ServiceRegistration.g.cs", SourceText.From(sourceBuilder.ToString(), Encoding.UTF8));
    }
}
```

### 3. Utilisation dans l'application principale

#### a) Ajouter le générateur au projet

Ajoutez une référence au projet Source Generator dans votre projet principal :

```bash
dotnet add reference path/to/SourceGenerator.csproj
```

et ajoutez également le `OutputItemType="Analyzer"` a la ligne de référence dans le fichier `.csproj` du projet principal :

```xml
    <ProjectReference Include="..\SourceGenerator\SourceGenerator.csproj" OutputItemType="Analyzer" />
```

#### b) Appeler les services générés

Dans votre application principale, utilisez la méthode générée :

```csharp
var builder = WebApplication.CreateBuilder(args);

// Appeler la méthode générée pour enregistrer les services
builder.Services.AddGeneratedServices();

var app = builder.Build();

app.Run();
```

### 4. Exemple de code généré

Le générateur produira un fichier similaire à celui-ci :

```csharp
// ServiceRegistration.g.cs
using Microsoft.Extensions.DependencyInjection;

namespace Generated
{
    public static class ServiceRegistration
    {
        public static IServiceCollection AddGeneratedServices(this IServiceCollection services)
        {
            services.AddTransient<IMyService, MyService>();
            services.AddTransient<IAnotherService, AnotherService>();
            return services;
        }
    }
}
```

### 5. Points importants

- **Performance** : Contrairement à la réflexion, le code généré est compilé et optimisé, ce qui améliore les performances.
- **Débogage** : Le code généré est visible dans le dossier `obj/Debug/netX/Generated` pour faciliter le débogage.
- **Personnalisation** : Vous pouvez ajouter des filtres ou des conventions pour contrôler quels services sont enregistrés.

### 6. Avantages par rapport à la réflexion

- **Rapidité** : Pas de surcharge liée à la réflexion au démarrage.
- **Sécurité** : Le code est généré à la compilation, ce qui réduit les erreurs d'exécution.
- **Lisibilité** : Le code généré est statique et peut être inspecté.

En utilisant un **Source Generator**, vous pouvez automatiser l'enregistrement des services tout en maintenant des performances élevées et un code maintenable.
