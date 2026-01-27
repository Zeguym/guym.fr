---
title: "Syntax API Roslyn pour les Source Generators (Code Gen)"
tags: dotnet roslyn source-generators syntax-api
series: "Source Generators en .NET"
---

La **Syntax API** de Roslyn est le point d'entrée bas niveau pour analyser le code C# dans un Source Generator. Elle permet de parcourir l'arbre de syntaxe (AST) des fichiers `.cs` pour repérer les éléments qui vous intéressent (attributs, classes, méthodes…) avant de générer du code.

Dans cet article, on va voir :

- ce qu'est un arbre de syntaxe ;
- comment utiliser un `ISyntaxReceiver` (générateurs classiques) ;
- comment retrouver les symboles à partir de la syntaxe ;
- comment aborder la Syntax API dans un générateur incrémental.

<!--more-->
{% include series.html %}
## 1. Rappel : Syntaxe vs Symboles

Roslyn sépare deux concepts :

- **Syntaxe** (`SyntaxTree`, `SyntaxNode`, `SyntaxToken`, `SyntaxTrivia`) : représentation textuelle du code (ce qui est écrit, avec la structure du langage).
- **Symboles** (`ISymbol`, `INamedTypeSymbol`, `IMethodSymbol`, etc.) : vue "sémantique" du code (types réels, héritage, interfaces implémentées…).

La Syntax API travaille côté **syntaxe** ; on complète ensuite avec la **Semantic Model API** pour obtenir les symboles.

## 2. Parcourir la syntaxe avec ISyntaxReceiver

Dans un générateur classique (`ISourceGenerator`), le point d'entrée typique pour la Syntax API est `ISyntaxReceiver`.

### 2.1. Enregistrer un SyntaxReceiver

Dans la méthode `Initialize` du générateur :

```csharp
[Generator]
public class DemoSyntaxGenerator : ISourceGenerator
{
	public void Initialize(GeneratorInitializationContext context)
	{
		context.RegisterForSyntaxNotifications(() => new DemoSyntaxReceiver());
	}

	public void Execute(GeneratorExecutionContext context)
	{
		if (context.SyntaxReceiver is not DemoSyntaxReceiver receiver)
			return;

		// Utiliser receiver pour générer du code…
	}
}
```

### 2.2. Implémenter le SyntaxReceiver

Un `ISyntaxReceiver` reçoit chaque nœud de syntaxe du projet ; on filtre ceux qui nous intéressent.

Exemple : collecter toutes les classes décorées par au moins un attribut :

```csharp
using System.Collections.Generic;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;

public sealed class DemoSyntaxReceiver : ISyntaxReceiver
{
	public List<ClassDeclarationSyntax> CandidateClasses { get; } = new();

	public void OnVisitSyntaxNode(SyntaxNode syntaxNode)
	{
		if (syntaxNode is ClassDeclarationSyntax { AttributeLists.Count: > 0 } cls)
		{
			CandidateClasses.Add(cls);
		}
	}
}
```

À ce stade, on travaille uniquement avec la **syntaxe** (pas encore les types réels).

## 3. De la syntaxe aux symboles

Dans `Execute`, on utilise la `Compilation` fournie par Roslyn pour récupérer un `SemanticModel` et obtenir un `INamedTypeSymbol` à partir de chaque `ClassDeclarationSyntax`.

```csharp
public void Execute(GeneratorExecutionContext context)
{
	if (context.SyntaxReceiver is not DemoSyntaxReceiver receiver)
		return;

	foreach (var classSyntax in receiver.CandidateClasses)
	{
		var semanticModel = context.Compilation.GetSemanticModel(classSyntax.SyntaxTree);
		if (semanticModel.GetDeclaredSymbol(classSyntax) is not INamedTypeSymbol typeSymbol)
			continue;

		// Exemple : vérifier la présence d'un attribut particulier
		var hasAttribute = typeSymbol.GetAttributes()
			.Any(a => a.AttributeClass?.Name == "GenerateMapperAttribute");

		if (!hasAttribute)
			continue;

		// À partir d'ici, on peut lire les propriétés, namespaces, etc.
		var properties = typeSymbol.GetMembers()
			.OfType<IPropertySymbol>()
			.Where(p => p.SetMethod is not null);

		// Génération de code basée sur ces informations…
	}
}
```

La Syntax API sert principalement à **réduire l'espace de recherche** (ne garder que les nœuds pertinents), puis la Semantic API donne les informations de type.

## 4. Travailler directement avec SyntaxNode / SyntaxTree

Parfois, on veut rester uniquement au niveau syntaxique, par exemple pour générer du code à partir de la structure exacte du fichier (commentaires, attributs, formes d'expressions).

Exemples typiques avec `ClassDeclarationSyntax` :

- `Identifier.Text` → nom de la classe.
- `BaseList` → classes de base / interfaces.
- `Members` → propriétés, méthodes, champs.

```csharp
foreach (var member in classSyntax.Members)
{
	if (member is PropertyDeclarationSyntax prop)
	{
		var propName = prop.Identifier.Text;
		var propType = prop.Type.ToString(); // vue syntaxique du type
		// …
	}
}
```

Ce niveau est utile si vous voulez reproduire fidèlement une forme de code ou analyser des patterns syntaxiques précis.

## 5. Syntax API et générateurs incrémentaux

Avec `IIncrementalGenerator`, on n'utilise plus `ISyntaxReceiver`, mais la logique reste basée sur la Syntax API, via les "providers".

Schéma simplifié :

```csharp
[Generator]
public class IncrementalDemoGenerator : IIncrementalGenerator
{
	public void Initialize(IncrementalGeneratorInitializationContext context)
	{
		var classDeclarations = context.SyntaxProvider
			.CreateSyntaxProvider(
				predicate: static (node, _) => node is ClassDeclarationSyntax { AttributeLists.Count: > 0 },
				transform: static (ctx, _) => (ClassDeclarationSyntax)ctx.Node)
			.Where(cls => cls is not null);

		context.RegisterSourceOutput(classDeclarations, (spc, cls) =>
		{
			// Ici, cls est un ClassDeclarationSyntax filtré par la Syntax API
			// On peut compléter avec SemanticModel si nécessaire (ctx.SemanticModel dans transform)
		});
	}
}
```

`CreateSyntaxProvider` fournit un moyen plus déclaratif de faire ce que `ISyntaxReceiver` faisait de manière impérative :

- `predicate` : filtre rapide basé sur la syntaxe.
- `transform` : conversion vers un modèle plus riche (syntaxique + sémantique si besoin).

## 6. Bonnes pratiques avec la Syntax API

- **Filtrer tôt** : plus vos prédicats syntaxiques sont précis, moins le générateur traite de nœuds inutiles.
- **Limiter les SemanticModel** : récupérer le `SemanticModel` est coûteux ; essayez de le faire le moins souvent possible et réutilisez‑le par `SyntaxTree` si nécessaire.
- **Ne pas dépendre des trivia** (espaces, commentaires) sauf si c'est intentionnel ; la mise en forme peut changer sans modifier la sémantique.
- **Toujours tolérer un code partiellement invalide** : le générateur doit rester robuste même quand le projet ne compile pas encore (code en cours de saisie).

La Syntax API est la porte d'entrée pour "voir" le code source dans un Source Generator. Bien maîtrisée, elle permet de cibler précisément ce que vous voulez analyser et de rester performant, tout en laissant la partie "compréhension des types" à la Semantic API.

