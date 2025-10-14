---
title: Tests d'architecture en .NET
tags: dotnet test architecture 
---

Les **tests d'architecture** en .NET permettent de valider que la structure du code respecte les règles et les conventions définies pour un projet. Ces tests sont particulièrement utiles pour garantir la maintenabilité et la cohérence du code, surtout dans des projets complexes.

<!--more-->

# Tests d'architecture

- **Respect des conventions** : Vérifier que les dépendances entre les couches (ex. : Domain, Application, Infrastructure) respectent les règles définies.
- **Éviter les mauvaises pratiques** : Empêcher l'utilisation de classes ou de bibliothèques non autorisées.
- **Faciliter la maintenance** : S'assurer que les développeurs respectent les principes d'architecture établis.

## Outils pour les tests d'architecture en .NET

1. **[NetArchTest](https://github.com/BenMorris/NetArchTest)**  
   Une bibliothèque légère pour écrire des tests d'architecture en .NET. Elle permet de valider les dépendances entre les namespaces, les classes, et les interfaces.

2. **[ArchUnitNET](https://github.com/TNG/ArchUnitNET)**  
   Inspiré de l'outil ArchUnit pour Java, il offre des fonctionnalités avancées pour tester les règles d'architecture.

## Exemple avec NetArchTest

### Installation

Ajoutez la bibliothèque NetArchTest à votre projet de tests :

```bash
dotnet add package NetArchTest.Rules
```

## Exemple de test d'architecture

Voici un exemple de test pour vérifier que les classes de la couche `Domain` ne dépendent pas de la couche `Infrastructure` :

```csharp
using NetArchTest.Rules;
using Xunit;

public class ArchitectureTests
{
    [Fact]
    public void Domain_Should_Not_Have_Dependencies_On_Infrastructure()
    {
        // Arrange & Act
        var result = Types.InAssembly(typeof(MyDomainClass).Assembly)
            .That()
            .ResideInNamespace("MyProject.Domain")
            .ShouldNot()
            .HaveDependencyOn("MyProject.Infrastructure")
            .GetResult();

        // Assert
        Assert.True(result.IsSuccessful, "Domain layer should not depend on Infrastructure layer.");
    }
}
```

**Ce que fait ce test :**

- Il analyse les types dans l'assembly contenant `MyDomainClass`.
- Il vérifie que les classes dans le namespace `MyProject.Domain` n'ont pas de dépendances vers `MyProject.Infrastructure`.
- Si une dépendance est détectée, le test échoue.

## Exemple avec ArchUnitNET

### Installation

Ajoutez la bibliothèque ArchUnitNET à votre projet de tests :

```bash
dotnet add package ArchUnitNET
```

Exemple de test
Voici un exemple similaire avec ArchUnitNET :

```csharp
using ArchUnitNET.Domain;
using ArchUnitNET.Fluent;
using ArchUnitNET.Loader;
using Xunit;

public class ArchitectureTests
{
    private static readonly Architecture Architecture =
        new ArchLoader().LoadAssemblies(typeof(MyDomainClass).Assembly).Build();

    [Fact]
    public void Domain_Should_Not_Depend_On_Infrastructure()
    {
        var domainLayer = ArchRuleDefinition.Types()
            .That().ResideInNamespace("MyProject.Domain")
            .Should().NotDependOnAny("MyProject.Infrastructure");

        domainLayer.Check(Architecture);
    }
}
```

### Intégration avec Husky

Vous pouvez configurer un hook Git pour exécuter vos tests d'architecture automatiquement avant un commit ou un push :

Exemple de hook `pre-push` :

```bash
dotnet husky add pre-push -c "dotnet test --filter Category=Architecture"
```

Pour utiliser ce filtre, vous pouvez marquer vos tests d'architecture avec un trait, par exemple :

```csharp
[Trait("Category", "Architecture")]
public class ArchitectureTests
{
    // ...
}
```

Cela garantit que les règles d'architecture sont respectées avant d'intégrer du code dans le dépôt.

# Différents types de tests d'architecture

Les tests d'architecture permettent de valider différents aspects de la structure et des règles d'un projet. Voici les principaux types de tests d'architecture que l'on peut réaliser :

## 1. Tests de dépendances entre couches

Ces tests vérifient que les dépendances entre les couches d'architecture respectent les règles définies. Par exemple :

- La couche **Domain** ne doit pas dépendre de la couche **Infrastructure**.
- La couche **Application** peut dépendre de **Domain**, mais pas de **Infrastructure**.

Exemple :

```csharp
Types.InAssembly(typeof(MyDomainClass).Assembly)
    .That().ResideInNamespace("MyProject.Domain")
    .ShouldNot().HaveDependencyOn("MyProject.Infrastructure")
    .GetResult();
```

## 2. Tests de respect des conventions de nommage

Ces tests s'assurent que les classes, interfaces, ou namespaces respectent les conventions de nommage définies. Par exemple :

- Les interfaces doivent commencer par `I`.
- Les classes de contrôleurs doivent se terminer par `Controller`.

Exemple :

```csharp
Types.InAssembly(typeof(MyController).Assembly)
    .That().AreClasses()
    .And().ResideInNamespace("MyProject.Controllers")
    .Should().HaveNameEndingWith("Controller")
    .GetResult();
```

## 3. Tests de dépendances externes

Ces tests vérifient que certaines bibliothèques ou frameworks externes ne sont pas utilisés dans des parties spécifiques du projet. Par exemple :

- La couche **Domain** ne doit pas dépendre d'Entity Framework ou d'ASP.NET Core.

Exemple :

```csharp
Types.InAssembly(typeof(MyDomainClass).Assembly)
    .That().ResideInNamespace("MyProject.Domain")
    .ShouldNot().HaveDependencyOn("Microsoft.EntityFrameworkCore")
    .GetResult();
```

## 4. Tests de structure des namespaces

Ces tests valident que les classes sont organisées dans les bons namespaces. Par exemple :

- Les classes de la couche **Infrastructure** doivent résider dans le namespace `MyProject.Infrastructure`.

Exemple :

```csharp
Types.InAssembly(typeof(MyInfrastructureClass).Assembly)
    .That().AreClasses()
    .Should().ResideInNamespace("MyProject.Infrastructure")
    .GetResult();
```

## 5. Tests de visibilité des classes

Ces tests vérifient que certaines classes ou types ont la bonne visibilité (public, internal, etc.). Par exemple :

- Dans certains contextes, vous pouvez décider que les classes d'une couche ne doivent pas être exposées en dehors d'un assembly (par exemple des types techniques internes à l'infrastructure).

Exemple :

```csharp
Types.InAssembly(typeof(MyInfrastructureClass).Assembly)
    .That().ResideInNamespace("MyProject.Infrastructure.Internal")
    .Should().BeInternal()
    .GetResult();
```

## 6. Tests de dépendances circulaires

Ces tests détectent les dépendances circulaires entre les classes ou les namespaces, qui peuvent rendre le code difficile à maintenir.

Exemple :
Avec ArchUnitNET, vous pouvez détecter les cycles entre namespaces :

```csharp
ArchRuleDefinition.Namespaces()
    .Should().NotHaveCyclicDependencies()
    .Check(Architecture);
```

## 7. Tests de règles spécifiques au projet

Ces tests sont personnalisés en fonction des besoins du projet. Par exemple :

- Vérifier que toutes les classes de services implémentent une interface.
- S'assurer que les classes de configuration sont marquées avec un attribut spécifique.

Exemple :

```csharp
Types.InAssembly(typeof(MyService).Assembly)
    .That().ResideInNamespace("MyProject.Services")
    .Should().ImplementInterface(typeof(IMyService))
    .GetResult();
```

## 8. Tests de conformité aux principes SOLID

Ces tests permettent de vérifier que le code respecte les principes SOLID.

### 1. Single Responsibility Principle (SRP)

Le principe de responsabilité unique stipule qu'une classe ne doit avoir qu'une seule raison de changer. Vous pouvez tester cela en vérifiant que les classes d'un namespace spécifique n'ont pas de dépendances inutiles.

```csharp
using NetArchTest.Rules;
using Xunit;

public class SRPTests
{
    [Fact]
    public void Services_Should_Not_Have_Dependencies_On_Controllers()
    {
        var result = Types.InAssembly(typeof(MyService).Assembly)
            .That().ResideInNamespace("MyProject.Services")
            .ShouldNot().HaveDependencyOn("MyProject.Controllers")
            .GetResult();

        Assert.True(result.IsSuccessful, "Services should not depend on Controllers to ensure SRP.");
    }
}
```

### 2. Open/Closed Principle (OCP)

Le principe OCP stipule que les classes doivent être ouvertes à l'extension mais fermées à la modification. Vous pouvez tester cela en vérifiant que les classes implémentent des interfaces ou utilisent des abstractions.

```csharp
using NetArchTest.Rules;
using Xunit;

public class OCPTests
{
    [Fact]
    public void Services_Should_Implement_Interfaces()
    {
        var result = Types.InAssembly(typeof(MyService).Assembly)
            .That().ResideInNamespace("MyProject.Services")
            .Should().ImplementInterface(typeof(IMyService))
            .GetResult();

        Assert.True(result.IsSuccessful, "All services should implement interfaces to respect OCP.");
    }
}
```

### 3. Liskov Substitution Principle (LSP)

Le principe LSP stipule que les classes dérivées doivent pouvoir être utilisées à la place de leurs classes de base sans altérer le comportement attendu. Vous pouvez tester cela en vérifiant que les classes dérivées respectent les contrats de leurs interfaces ou classes de base.

Exemple avec ArchUnitNET :

```csharp
using ArchUnitNET.Domain;
using ArchUnitNET.Fluent;
using ArchUnitNET.Loader;
using Xunit;

public class LSPTests
{
    private static readonly Architecture Architecture =
        new ArchLoader().LoadAssemblies(typeof(MyBaseClass).Assembly).Build();

    [Fact]
    public void DerivedClasses_Should_Not_Violate_BaseClass_Contracts()
    {
        var rule = ArchRuleDefinition.Types()
            .That().AreAssignableTo(typeof(MyBaseClass))
            .Should().BeAssignableTo(typeof(MyBaseClass));

        rule.Check(Architecture);
    }
}
```
En pratique, il est difficile de tester automatiquement tous les aspects de LSP avec des outils d'architecture : on se repose surtout sur les tests fonctionnels et unitaires pour s'assurer que les classes dérivées respectent les contrats de leurs classes de base.

### 4. Interface Segregation Principle (ISP)

Le principe ISP stipule qu'une classe ne doit pas être forcée d'implémenter des interfaces qu'elle n'utilise pas. Vous pouvez tester cela en vérifiant que les interfaces ne sont pas trop larges.

```csharp
using NetArchTest.Rules;
using Xunit;

public class ISPTests
{
    [Fact]
    public void Interfaces_Should_Not_Be_Too_Broad()
    {
        var result = Types.InAssembly(typeof(IMyInterface).Assembly)
            .That().AreInterfaces()
            .ShouldNot().HaveDependencyOn("System.Collections.Generic.List`1") // Exemple : éviter des dépendances inutiles
            .GetResult();

        Assert.True(result.IsSuccessful, "Interfaces should not force implementations to depend on unused types.");
    }
}
```
Les outils de tests d'architecture peuvent aider indirectement en vérifiant, par exemple, que les interfaces d'une couche ne dépendent pas de types techniques d'autres couches.

### 5. Dependency Inversion Principle (DIP)

Le principe DIP stipule que les modules de haut niveau ne doivent pas dépendre des modules de bas niveau, mais des abstractions. Vous pouvez tester cela en vérifiant que les classes de haut niveau dépendent uniquement d'interfaces ou d'abstractions.

```csharp
using NetArchTest.Rules;
using Xunit;

public class DIPTests
{
    [Fact]
    public void HighLevelModules_Should_Depend_On_Abstractions_Only()
    {
        var result = Types.InAssembly(typeof(MyHighLevelClass).Assembly)
            .That().ResideInNamespace("MyProject.Application")
            .Should()
            .HaveDependencyOn("MyProject.Domain.Abstractions")
            .AndShouldNot()
            .HaveDependencyOn("MyProject.Infrastructure")
            .GetResult();

        Assert.True(result.IsSuccessful, "High-level modules should depend on abstractions, not concrete infrastructure implementations.");
    }
}
```
Les outils de tests d'architecture peuvent aider indirectement en vérifiant, par exemple, que les interfaces d'une couche ne dépendent pas de types techniques d'autres couches.

### **Résumé des tests SOLID**

| Principe | Objectif du test                                                               | Exemple                                                               |
| -------- | ------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| **SRP**  | Vérifier que les classes n'ont qu'une seule responsabilité.                    | Les services ne doivent pas dépendre des contrôleurs.                 |
| **OCP**  | Vérifier que les classes sont extensibles via des interfaces ou abstractions.  | Les services doivent implémenter des interfaces.                      |
| **LSP**  | Vérifier que les classes dérivées respectent les contrats des classes de base. | Les classes dérivées doivent être assignables à leurs bases.          |
| **ISP**  | Vérifier que les interfaces ne sont pas trop larges.                           | Les interfaces ne doivent pas forcer des dépendances inutiles.        |
| **DIP**  | Vérifier que les modules de haut niveau dépendent des abstractions.            | Les modules applicatifs doivent dépendre des abstractions du domaine. |

Ces tests permettent de garantir que votre code respecte les principes SOLID, ce qui améliore sa maintenabilité, sa flexibilité et sa qualité globale.
