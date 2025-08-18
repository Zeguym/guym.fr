---
title: Optimiser les performances avec EF Core
tags: dotnet efcore performance compiled-query cache dbcontext-pool
series: "Entity Framework"
---

Entity Framework Core : accélérer les accès avec les requêtes précompilées, la mise en cache et le DbContext pooling

<!--more-->
{% include series.html %}

# 1. Requêtes précompilées

## 1.1. Principe

Par défaut, EF Core analyse l'expression LINQ puis la traduit en SQL.
Quand une requête est exécutée très souvent, il est possible de compiler cette requête une fois, puis de réutiliser le délégué compilé.

## 1.2. Compiled vs precompiled queries (et selon la version d'EF)

Le vocabulaire peut porter à confusion, car on voit souvent les deux termes.

- Compiled query : c'est le terme officiel des API EF Core avec `EF.CompileQuery` / `EF.CompileAsyncQuery`.
- Precompiled query : c'est souvent utilisé comme synonyme dans les articles, mais techniquement cela peut aussi désigner une génération anticipée au build (AOT) dans les versions récentes.

En pratique, retenez ceci :

- EF6 (Entity Framework "classique") : on utilisait `CompiledQuery.Compile(...)` et le gain pouvait être significatif sur des requêtes très répétées.
- EF Core 1 à 7 : EF Core met déjà en cache une partie du pipeline de traduction. Les compiled queries explicites existent toujours et servent surtout sur les hot paths.
- EF Core 8+ : on garde `EF.CompileQuery` pour les scénarios classiques, et on peut aussi rencontrer la notion de precompiled queries dans le contexte Native AOT/source generation.

Conclusion rapide : dans la plupart des projets, quand on dit "requête précompilée" en EF Core, on parle généralement des compiled queries via `EF.CompileQuery`.

## 1.3. Exemple

```csharp
using Microsoft.EntityFrameworkCore;

public static class ProductQueries
{
	// Requête asynchrone compilée et réutilisable
	public static readonly Func<AppDbContext, int, IAsyncEnumerable<ProductDto>>
		GetActiveProductById = EF.CompileAsyncQuery(
			(AppDbContext db, int id) =>
				db.Products
					.AsNoTracking()
					.Where(p => p.Id == id && p.IsActive)
					.Select(p => new ProductDto(p.Id, p.Name, p.Price))
		);
}

public sealed record ProductDto(int Id, string Name, decimal Price);
```

Utilisation :

```csharp
var dto = await ProductQueries
	.GetActiveProductById(context, productId)
	.FirstOrDefaultAsync(cancellationToken);
```

## 1.4. Quand l'utiliser

- Requête très fréquente et stable (même forme, seuls les paramètres changent).
- Endpoints à fort trafic.
- Scénarios de lecture où la latence est critique.

## 1.5. Limites

- Gain faible pour les requêtes occasionnelles.
- Complexité supplémentaire si la requête évolue souvent.
- La compilation aide surtout le pipeline EF, pas une requête SQL lente et mal indexée.

# 2. Mise en cache applicative

## 2.1. Principe

Si certaines données changent peu, inutile d'interroger la base à chaque appel.
On peut utiliser IMemoryCache pour les lectures rapides en local (ou un cache distribué pour plusieurs instances).

## 2.2. Exemple avec IMemoryCache

```csharp
using Microsoft.Extensions.Caching.Memory;

public class ProductReadService
{
	private readonly AppDbContext _db;
	private readonly IMemoryCache _cache;

	public ProductReadService(AppDbContext db, IMemoryCache cache)
	{
		_db = db;
		_cache = cache;
	}

	public async Task<ProductDto?> GetByIdAsync(int id, CancellationToken ct)
	{
		var cacheKey = $"product:{id}";

		if (_cache.TryGetValue(cacheKey, out ProductDto? cached))
		{
			return cached;
		}

		var dto = await _db.Products
			.AsNoTracking()
			.Where(p => p.Id == id && p.IsActive)
			.Select(p => new ProductDto(p.Id, p.Name, p.Price))
			.FirstOrDefaultAsync(ct);

		if (dto is null)
		{
			return null;
		}

		_cache.Set(cacheKey, dto, new MemoryCacheEntryOptions
		{
			SlidingExpiration = TimeSpan.FromMinutes(5),
			AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
		});

		return dto;
	}
}
```

## 2.3. Bonnes pratiques cache

- Mettre en cache des DTO, pas des entités suivies par le Change Tracker.
- Définir une stratégie d'invalidation claire (TTL, événement, suppression explicite).
- Éviter un TTL trop long sur des données métier sensibles.
- Mesurer le ratio hit/miss pour valider le gain.

# 3. DbContext pooling

## 3.1. Principe

Créer un DbContext à chaque requête a un coût.
Le pooling permet de réutiliser des instances de contexte réinitialisées, ce qui réduit les allocations et la pression du GC.

## 3.2. Configuration

```csharp
builder.Services.AddDbContextPool<AppDbContext>(options =>
{
	options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"));
});
```

## 3.3. Points d'attention

- Un DbContext reste non thread-safe : une instance, un flux d'exécution.
- Ne pas stocker d'état métier mutable dans le contexte.
- Préférer des traitements courts par unité de travail.
- Vérifier les comportements spécifiques si vous injectez des services dépendants du contexte.

# 4. Combiner les 3 leviers

La combinaison la plus efficace en lecture fréquente :

1. cache en premier niveau,
2. requête compilée en fallback,
3. contexte poolé pour limiter le coût de création.

Exemple de logique :

```csharp
public async Task<ProductDto?> GetProductFastAsync(int id, CancellationToken ct)
{
	var key = $"product:{id}";

	if (_cache.TryGetValue(key, out ProductDto? dto))
	{
		return dto;
	}

	dto = await ProductQueries
		.GetActiveProductById(_db, id)
		.FirstOrDefaultAsync(ct);

	if (dto is not null)
	{
		_cache.Set(key, dto, TimeSpan.FromMinutes(10));
	}

	return dto;
}
```

# 5. Checklist performance EF Core

- Requêtes de lecture en AsNoTracking().
- Projections DTO via `Select` pour limiter les colonnes.
- Requêtes précompilées seulement sur les hot paths.
- Cache avec invalidation explicite.
- AddDbContextPool pour les applications à charge soutenue.
- Index SQL alignés avec vos filtres et vos tris.
- Mesure réelle avec logs, traces et benchmarks avant/après.

# Conclusion

Les performances EF Core ne reposent pas sur un seul bouton magique.
Le bon résultat vient de la combinaison :

- requêtes efficaces,
- cache maîtrisé,
- cycle de vie du contexte optimisé.

Commencez par mesurer vos endpoints les plus lents, appliquez ces trois leviers sur les chemins critiques, puis validez le gain en production avec de la télémétrie.
