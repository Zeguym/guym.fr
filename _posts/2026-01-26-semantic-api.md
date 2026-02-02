---
title: "Semantic API Roslyn pour les Source Generators (Code Gen)"
tags: dotnet roslyn source-generators semantic-api
series: "Source Generators en .NET"
---

La **Semantic API** de Roslyn complète la Syntax API en donnant accès à la compréhension "sémantique" du code : types réels, symboles, héritage, interfaces implémentées, attributs résolus, etc. Dans un Source Generator, c'est elle qui permet de passer d'un simple nœud de syntaxe à un modèle riche sur lequel baser la génération de code.

Dans cet article, on va voir :

- ce qu'est un `SemanticModel` ;
- comment obtenir des `ISymbol` (types, méthodes, propriétés) à partir de la syntaxe ;
- comment lire les attributs et types effectifs ;
- comment utiliser efficacement la Semantic API dans un générateur incrémental.

<!--more-->
{% include series.html %}

## 1. SemanticModel : le lien entre syntaxe et symboles

La `Compilation` fournie à un source generator contient toute la connaissance du compilateur sur le projet. À partir d'elle, on peut récupérer un `SemanticModel` pour un `SyntaxTree` donné :

```csharp
var semanticModel = context.Compilation.GetSemanticModel(classSyntax.SyntaxTree);
```

Le `SemanticModel` permet par exemple de :

- obtenir le symbole déclaré par un nœud (`GetDeclaredSymbol`) ;
- interroger le type d'une expression (`GetTypeInfo`) ;
- résoudre le symbole référencé par un identifiant (`GetSymbolInfo`).

Dans un Source Generator, on l'utilise généralement pour transformer un `SyntaxNode` filtré (via la Syntax API) en `ISymbol` exploitable.

## 2. Récupérer les symboles depuis la syntaxe

Partons du receveur syntaxique de l'article précédent, qui collecte des `ClassDeclarationSyntax`. Dans `Execute`, on veut maintenant récupérer des `INamedTypeSymbol` pour ces classes.

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

		// À partir d'ici, on travaille avec le symbole, pas seulement avec la syntaxe
		var className = typeSymbol.Name;                 // nom simple
		var fullName = typeSymbol.ToDisplayString();    // nom complet avec namespace
	}
}
```

Le symbole `INamedTypeSymbol` offre une vue riche :

- `ContainingNamespace`, `BaseType`, `AllInterfaces` ;
- `GetMembers()` pour accéder aux propriétés, méthodes, champs, etc.

## 3. Lire les propriétés et leurs types

Reprenons le cas du générateur de mapper : on veut projeter toutes les propriétés "mappables" d'un type.

```csharp
var properties = typeSymbol.GetMembers()
	.OfType<IPropertySymbol>()
	.Where(p => p.SetMethod is not null); // seulement les propriétés avec setter

foreach (var prop in properties)
{
	var propName = prop.Name;
	var propType = prop.Type.ToDisplayString();
	var isNullable = prop.NullableAnnotation == NullableAnnotation.Annotated;

	// Utiliser ces infos pour générer du code
}
```

Grâce à la Semantic API, `prop.Type` est un vrai symbole de type (`ITypeSymbol`), pas juste une chaîne de caractères issue de la syntaxe.

## 4. Lire et interpréter les attributs

Les attributs sont une des principales sources de configuration pour un Source Generator. Avec la Semantic API, on peut :

- vérifier la présence d'un attribut donné ;
- lire ses arguments (nommés et positionnels) ;
- récupérer les types passés en paramètres.

Exemple avec un attribut `[GenerateMapper(typeof(CustomerDto))]` :

```csharp
var mapperAttributes = typeSymbol.GetAttributes()
	.Where(a => a.AttributeClass?.Name == "GenerateMapperAttribute");

foreach (var attr in mapperAttributes)
{
	// Argument positionnel 0 : typeof(CustomerDto)
	if (attr.ConstructorArguments.FirstOrDefault().Value is INamedTypeSymbol targetType)
	{
		var targetTypeName = targetType.ToDisplayString();
		// targetType fournit aussi ses propres propriétés, namespace, etc.
	}
}
```

C'est ce mécanisme qui permet de passer de simples annotations dans le code utilisateur à une configuration riche côté générateur.

## 5. Semantic API dans un générateur incrémental

Avec `IIncrementalGenerator`, on évite généralement d'appeler `GetSemanticModel` directement dans la boucle principale. À la place, on enrichit les données dès la phase `transform` du `SyntaxProvider`.

Schéma simplifié :

```csharp
[Generator]
public class IncrementalSemanticDemo : IIncrementalGenerator
{
	public void Initialize(IncrementalGeneratorInitializationContext context)
	{
		var candidates = context.SyntaxProvider
			.CreateSyntaxProvider(
				predicate: static (node, _) => node is ClassDeclarationSyntax { AttributeLists.Count: > 0 },
				transform: static (ctx, _) =>
				{
					var classSyntax = (ClassDeclarationSyntax)ctx.Node;
					var symbol = (INamedTypeSymbol?)ctx.SemanticModel.GetDeclaredSymbol(classSyntax);
					return symbol;
				})
			.Where(symbol => symbol is not null)!;

		context.RegisterSourceOutput(candidates, (spc, typeSymbol) =>
		{
			// Ici, on a déjà l'INamedTypeSymbol : plus besoin de SemanticModel
			// Génération de code à partir du symbole…
		});
	}
}
```

L'idée :

- utiliser la Syntax API pour filtrer vite (dans `predicate`) ;
- utiliser la Semantic API dans `transform` pour produire directement des symboles ;
- ne conserver dans le pipeline que les données dont on a besoin pour la génération.

## 6. Bonnes pratiques avec la Semantic API

- **Limiter les appels à SemanticModel** : c'est une opération coûteuse. Regroupez si possible les requêtes par `SyntaxTree` et privilégiez l'utilisation de `ctx.SemanticModel` dans les générateurs incrémentaux.
- **Travailler le plus possible avec les symboles** : une fois les symboles obtenus, faites vos filtrages/transformations dessus plutôt que de revenir à la syntaxe.
- **Utiliser `ToDisplayString` avec un format contrôlé** (`SymbolDisplayFormat`) si vous avez besoin de noms stables (pour des clés de cache, des diagnostics, etc.).
- **Gérer le cas du code invalide** : certains symboles peuvent être partiels ou manquants pendant l'édition, prévoyez des `null` et des `continue` défensifs.

La Semantic API est le cœur "intelligent" d'un Source Generator : c'est elle qui transforme des nœuds de syntaxe filtrés en une vue riche du modèle .NET (types, membres, attributs). Combinée à la Syntax API pour cibler efficacement ce qui vous intéresse, elle permet de générer un code précis, fiable et aligné sur la réalité du projet compilé.

