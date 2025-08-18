---
title: "Source Generators en .NET (Code Gen)"
tags: dotnet roslyn source-generators performance outillage
series: "Source Generators en .NET"
---

Les **Source Generators** sont une fonctionnalité introduite avec .NET 5 et C# 9 qui permet de générer du code C# à la compilation, à partir du code existant. Contrairement aux outils de génération de code "hors bande" (T4, scripts, etc.), les source generators s'intègrent directement au compilateur Roslyn et fonctionnent de manière transparente dans Visual Studio, Rider ou VS Code.

Ils sont particulièrement utiles pour :

- Éviter le code répétitif (boilerplate).
- Générer du code optimisé à partir de métadonnées (attributs, fichiers JSON, etc.).
- Fournir de l'intellisense et des diagnostics au moment de l'écriture du code.

<!--more-->
{% include series.html %}
## 1. Compatibilité et périmètre

Les source generators fonctionnent avec les projets **SDK-style** basés sur le compilateur Roslyn, en pratique :

- Projets ciblant **.NET 5 et versions ultérieures** (net5.0, net6.0, net7.0, net8.0, …) avec **C# 9 ou plus récent**.
- Projets **.NET Core 3.1** ou **.NET Standard** consommés depuis un projet .NET 5+ qui exécute le générateur.
- Un générateur lui‑même est généralement compilé en **netstandard2.0** pour être utilisable depuis différents frameworks.

En revanche, les anciens projets **.NET Framework au format non‑SDK** ne supportent pas nativement les source generators : il est recommandé de migrer vers un projet SDK (ou vers .NET moderne) pour en profiter pleinement.

## 2. Principe général

Un source generator est une extension du compilateur Roslyn qui :

1. Observe l'arbre de syntaxe du projet (les fichiers `.cs`).
2. Analyse certains symboles (attributs, interfaces, classes…).
3. Génère dynamiquement du nouveau code C# qui sera compilé avec le reste du projet.

Important :

- Le générateur **ne modifie pas** vos fichiers existants : il ajoute des fichiers générés en mémoire (ou dans `obj/` pour inspection).
- Vous pouvez voir le code généré dans Visual Studio (dossier *Analyzers* → *Source Generators* → fichiers générés) ou dans Rider.
- Le code généré est compilé comme n'importe quel autre fichier source.

## 3. Quand utiliser un Source Generator ?

### 3.1. Quelques scénarios typiques en .NET :

- **Sérialisation / désérialisation** : générer du code de sérialisation optimisé (par exemple `System.Text.Json` avec `JsonSerializerContext`).
- **Mapping** : générer des mappers entre DTOs et entités (au lieu d'écrire toujours les mêmes `MyDto.ToEntity()`).
- **Validation** : générer du code de validation à partir d'attributs.
- **Configuration / IOptions** : générer des classes strongly-typed à partir de fichiers de configuration.
- **Client HTTP / API** : générer des clients typés à partir d'annotations ou de contrats.

L'idée est toujours la même : remplacer du code répétitif, sujet aux erreurs, par du code généré de façon fiable à la compilation.

### 3.2. Par rapport à la réflexion classique

On pourrait faire une partie de ces scénarios avec de la **réflexion** (inspection de types à l'exécution, `Activator.CreateInstance`, etc.), mais les source generators apportent plusieurs avantages importants :

- **Performances** : avec la réflexion, le travail d'analyse des types se fait à l'exécution, avec un coût non négligeable (découverte des membres, allocations, caches à maintenir). Avec un source generator, tout ce travail est fait **à la compilation** et le code généré est du C# "normal", aussi rapide qu'un code écrit à la main.
- **Compatibilité AOT / trimming** : en Native AOT ou avec un trimming agressif, la réflexion devient fragile (types supprimés, besoin d'annoter ce qui doit être conservé). Les source generators, eux, produisent du code statique, beaucoup plus simple à faire fonctionner dans ces contextes.
- **Sécurité de typage et IDE** : la réflexion repose souvent sur des chaînes de caractères (noms de propriétés, de types) et les erreurs apparaissent au runtime. Avec un générateur, le code résultant est typé, compilé, avec IntelliSense, navigation, diagnostics Roslyn et erreurs de compilation classiques.
- **Debuggabilité** : la logique basée sur la réflexion est souvent "cachée" derrière des APIs génériques. Avec un source generator, on peut ouvrir les fichiers `.g.cs`, voir exactement ce qui est appelé et profiler/déboguer comme du code habituel.
- **Contrats plus explicites** : la réflexion encourage parfois des designs très permissifs. Les générateurs poussent à définir des contrats clairs (attributs, interfaces, conventions), et à refuser la génération (en signalant un diagnostic) en cas de configuration incorrecte.

La réflexion reste utile quand on a besoin de **découverte vraiment dynamique** (plugins, scripts, types inconnus à la compilation). Les source generators brillent quand on connaît le modèle à la compilation et qu'on veut un code optimisé, AOT‑friendly et sûr.

| Aspect                        | Réflexion classique                             | Source generators                                         |
| ----------------------------- | ----------------------------------------------- | --------------------------------------------------------- |
| Moment où le travail est fait | À l'exécution                                   | À la compilation                                          |
| Coût en performance           | Plus élevé (découverte + allocations)           | Équivalent à du code écrit à la main                     |
| Compatibilité AOT/trimming    | Fragile sans annotations spécifiques            | Très bonne (code statique généré)                        |
| Typage / IDE                  | Chaînes de caractères, erreurs au runtime       | Code typé, erreurs de compilation, IntelliSense complet  |
| Debug / profiling             | Logique souvent cachée derrière des APIs génériques | Fichiers `.g.cs` visibles et déboguables comme le reste  |
| Dynamisme                     | Forte (types inconnus à la compilation)         | Faible (modèle connu à la compilation requis)            |

## 4. Créer un Source Generator simple

Un source generator est une **bibliothèque de classes** qui référence les packages Roslyn et implémente l'interface `ISourceGenerator` (ou dérive de `IIncrementalGenerator` dans les versions récentes).

### 4.1. Créer le projet de générateur

Dans un dossier séparé (solution de préférence à plusieurs projets) :

```bash
dotnet new classlib -n Demo.SourceGenerators
cd Demo.SourceGenerators
dotnet add package Microsoft.CodeAnalysis.CSharp
dotnet add package Microsoft.CodeAnalysis.Analyzers
```

Ensuite, dans le `.csproj` du générateur, on indique qu'il s'agit d'un **analyzer avec générateur** :

```xml
<Project Sdk="Microsoft.NET.Sdk">
	<PropertyGroup>
		<TargetFramework>netstandard2.0</TargetFramework>
		<IncludeBuildOutput>false</IncludeBuildOutput>
	</PropertyGroup>

	<ItemGroup>
		<PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="*" PrivateAssets="all" />
		<PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="*" PrivateAssets="all" />
	</ItemGroup>

	<ItemGroup>
		<Analyzer Include="$(TargetPath)" />
	</ItemGroup>
</Project>
```

> En pratique, on fixe des versions précises plutôt que `*`, mais cela suffit pour illustrer la structure.

### 4.2. Implémenter un générateur minimal

On crée une classe qui implémente `ISourceGenerator` :

```csharp
using System.Text;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.Text;

namespace Demo.SourceGenerators;

[Generator]
public class HelloGenerator : ISourceGenerator
{
		public void Initialize(GeneratorInitializationContext context)
		{
				// Optionnel : enregistrement de callbacks, diagnostics, etc.
		}

		public void Execute(GeneratorExecutionContext context)
		{
				const string source = @"namespace Demo.Generated
{
		public static class HelloFromGenerator
		{
				public static string Message => \"Hello from source generator!\";
		}
}";

				context.AddSource(
						"HelloFromGenerator.g.cs",
						SourceText.From(source, Encoding.UTF8));
		}
}
```

Ce générateur ajoute un fichier `HelloFromGenerator.g.cs` au projet consommateur, avec une classe simple.

### 4.3. Exemple métier : générer un mapper

Prenons un exemple un peu plus proche d'un cas réel : générer du code de mapping entre un type `Entity` et un type `Dto` à partir d'un attribut.

On commence par définir un attribut dans le projet consommateur :

```csharp
namespace Demo.App;

[AttributeUsage(AttributeTargets.Class, AllowMultiple = true)]
public class GenerateMapperAttribute : Attribute
{
	public Type TargetType { get; }

	public GenerateMapperAttribute(Type targetType)
	{
		TargetType = targetType;
	}
}
```

On peut ensuite annoter nos modèles :

```csharp
namespace Demo.App;

[GenerateMapper(typeof(CustomerDto))]
public class Customer
{
	public int Id { get; set; }
	public string Name { get; set; } = string.Empty;
}

public class CustomerDto
{
	public int Id { get; set; }
	public string Name { get; set; } = string.Empty;
}
```

Le générateur (simplifié) va scanner les classes décorées par `GenerateMapperAttribute` et produire une méthode de mapping très basique :

```csharp
using System.Linq;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Text;
using System.Text;

namespace Demo.SourceGenerators;

[Generator]
public class MapperGenerator : ISourceGenerator
{
	public void Initialize(GeneratorInitializationContext context)
	{
		context.RegisterForSyntaxNotifications(() => new MapperSyntaxReceiver());
	}

	public void Execute(GeneratorExecutionContext context)
	{
		if (context.SyntaxReceiver is not MapperSyntaxReceiver receiver)
			return;

		foreach (var candidate in receiver.Candidates)
		{
			var model = context.Compilation.GetSemanticModel(candidate.SyntaxTree);
			var symbol = model.GetDeclaredSymbol(candidate) as INamedTypeSymbol;
			if (symbol is null)
				continue;

			var mapperAttributes = symbol.GetAttributes()
				.Where(a => a.AttributeClass?.Name == "GenerateMapperAttribute");

			foreach (var attr in mapperAttributes)
			{
				if (attr.ConstructorArguments.FirstOrDefault().Value is not INamedTypeSymbol targetType)
					continue;
				var sourceName = symbol.ToDisplayString();
				var targetName = targetType.ToDisplayString();
{% raw %}
				var source = $@"namespace Demo.App.Generated
					{{
						public static class {symbol.Name}Mapper
						{{ 
							public static {targetName} ToDto({sourceName} source)
								=> new {targetName}
								{{
									Id = source.Id,
									Name = source.Name
								}};
						}}
					}}";
{% endraw %}
				context.AddSource($"{symbol.Name}Mapper.g.cs", SourceText.From(source, Encoding.UTF8));
			}
		}
	}

	private sealed class MapperSyntaxReceiver : ISyntaxReceiver
	{
		public List<ClassDeclarationSyntax> Candidates { get; } = new();

		public void OnVisitSyntaxNode(SyntaxNode syntaxNode)
		{
			if (syntaxNode is ClassDeclarationSyntax { AttributeLists.Count: > 0 } cls)
			{
				Candidates.Add(cls);
			}
		}
	}
}
```

Dans le code applicatif, vous pouvez ensuite utiliser directement le mapper généré :

```csharp
using Demo.App;
using Demo.App.Generated;

var customer = new Customer { Id = 1, Name = "Alice" };
CustomerDto dto = CustomerMapper.ToDto(customer);
```

Cet exemple reste volontairement simple (pas de gestion de propriétés manquantes, pas de collections, etc.), mais il illustre bien un cas d'usage typique : **remplacer du mapping manuel répétitif par du code généré à la compilation**.

## 5. Consommer le Source Generator dans un projet .NET

Dans un second projet (par exemple une appli console) :

```bash
dotnet new console -n Demo.App
cd Demo.App
dotnet add reference ../Demo.SourceGenerators/Demo.SourceGenerators.csproj
```

Dans le `.csproj` du projet consommateur, on référence le projet de générateur comme **analyzer** :

```xml
<Project Sdk="Microsoft.NET.Sdk">
	<PropertyGroup>
		<OutputType>Exe</OutputType>
		<TargetFramework>net8.0</TargetFramework>
	</PropertyGroup>

	<ItemGroup>
		<ProjectReference Include="..\Demo.SourceGenerators\Demo.SourceGenerators.csproj" 
											OutputItemType="Analyzer" 
											ReferenceOutputAssembly="false" />
	</ItemGroup>
</Project>
```

Vous pouvez ensuite utiliser le code généré dans le projet console :

```csharp
using System;
using Demo.Generated;

Console.WriteLine(HelloFromGenerator.Message);
```

En construisant le projet, le générateur s'exécute, produit la classe `HelloFromGenerator`, puis le compilateur compile l'ensemble.

## 6. Source Generators incrémentaux (recommandés)

Les premières versions de générateurs (`ISourceGenerator`) fonctionnent, mais les générateurs **incrémentaux** (`IIncrementalGenerator`) sont généralement recommandés pour :

- Améliorer les performances (recalcul uniquement des parties impactées par une modification).
- Simplifier la logique en déclarant une "pipeline" (inputs → transform → outputs).

L'idée :

- Déclarer quelles parties du code vous intéressent (par exemple toutes les classes décorées par un attribut `[AutoNotify]`).
- Projeter ces symboles vers un modèle plus simple.
- Générer le code pour chaque élément.

L'API est plus déclarative, mais l'idée reste la même : observer → transformer → générer.

## 7. Bonnes pratiques

- **Limiter ce qui est généré** : éviter de générer des milliers de lignes inutiles.
- **Fournir de bons diagnostics** : utiliser `context.ReportDiagnostic` pour guider le développeur en cas d'erreur de configuration.
- **Documenter le code généré** : même s'il est technique, quelques commentaires ou une doc dans le README du package aident à l'adoption.
- **Ne pas tout résoudre avec des générateurs** : parfois, un simple modèle de code, un enregistrement (`record`) ou une méthode d'extension suffisent.

## 8. Exemples de générateurs existants

Plutôt que de tout réinventer, il est souvent intéressant d'utiliser des générateurs déjà existants :

- `System.Text.Json` : générateurs de contexte de sérialisation (`JsonSerializerContext`).
- Générateurs pour `IOptions` / configuration dans ASP.NET Core.
- Bibliothèques communautaires pour le mapping, la validation, l'implémentation d'INotifyPropertyChanged, etc.

Ces projets sont de bonnes sources d'inspiration pour concevoir vos propres générateurs.

## 9. Conclusion

Les Source Generators apportent une nouvelle façon de factoriser et d'optimiser le code en .NET, en déplaçant une partie du travail au moment de la compilation. Utilisés avec parcimonie et bien documentés, ils permettent de réduire le code répétitif, d'améliorer les performances et d'offrir une meilleure expérience de développement (intellisense, diagnostics, découverte du code généré).

Commencez par un générateur très simple, qui ajoute une classe ou une méthode utilitaire, puis faites-le évoluer vers un cas plus métier (mapping, validation, configuration) au fur et à mesure que vous vous familiarisez avec l'API Roslyn.

