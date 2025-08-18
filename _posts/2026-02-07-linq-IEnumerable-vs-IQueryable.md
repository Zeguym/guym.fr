---
title: "IEnumerable<T> vs IQueryable<T> : deux mondes, un même LINQ"
tags: dotnet basics linq csharp ienumerable iqueryable
series: "Linq"
---

Quand on écrit `.Where(x => x.IsActive)`, la syntaxe est identique que l'on travaille sur une `List<T>` ou sur un `DbSet<T>` Entity Framework. Pourtant le comportement est radicalement différent : d'un côté le filtre s'exécute en mémoire, de l'autre il est traduit en SQL. Comprendre cette dualité `IEnumerable<T>` / `IQueryable<T>` est essentiel pour éviter les problèmes de performance et les bugs subtils.

<!--more-->
{% include series.html %}

# Deux interfaces, deux stratégies

## `IEnumerable<T>` — exécution en mémoire

`IEnumerable<T>` est l'interface de base de toute séquence en .NET. Les opérateurs LINQ définis dans `System.Linq.Enumerable` travaillent sur cette interface.

```csharp
// Les opérateurs reçoivent des delegates (Func<T, bool>)
public static IEnumerable<T> Where<T>(
    this IEnumerable<T> source,
    Func<T, bool> predicate);
```

Le prédicat est un **delegate** : du code compilé en IL, exécuté directement par le CLR en mémoire.

## `IQueryable<T>` — traduction par un provider

`IQueryable<T>` hérite de `IEnumerable<T>` mais les opérateurs LINQ sont définis dans `System.Linq.Queryable` et reçoivent des **arbres d'expressions**.

```csharp
// Les opérateurs reçoivent des arbres d'expressions (Expression<Func<T, bool>>)
public static IQueryable<T> Where<T>(
    this IQueryable<T> source,
    Expression<Func<T, bool>> predicate);
```

Le prédicat est un **arbre d'expression** (`Expression<Func<T, bool>>`) : une structure de données qui **décrit** le code sans l'exécuter. Un provider (Entity Framework, LINQ to SQL, etc.) peut l'analyser et le traduire dans un autre langage (SQL, HTTP, etc.).

# `Func<T, bool>` vs `Expression<Func<T, bool>>`

C'est **la** différence fondamentale.

## `Func<T, bool>` — du code exécutable

```csharp
Func<int, bool> estPair = x => x % 2 == 0;

// C'est un delegate : on peut l'appeler
bool resultat = estPair(42); // true
```

Le compilateur transforme la lambda en **méthode anonyme** compilée en IL. C'est une boîte noire : on peut l'exécuter, mais pas l'inspecter.

## `Expression<Func<int, bool>>` — une description du code

```csharp
Expression<Func<int, bool>> estPair = x => x % 2 == 0;

// C'est un arbre : on peut l'analyser
// estPair.Body → (x % 2) == 0
// estPair.Body.NodeType → Equal
// estPair.Body.Left → (x % 2)
// estPair.Body.Left.Left → x
// estPair.Body.Left.Right → 2
```

Le compilateur génère un **arbre de syntaxe** qu'on peut parcourir, transformer et traduire. C'est ce qu'Entity Framework fait pour produire du SQL.

```
         Equal (==)
        /          \
    Modulo (%)      Constant (0)
    /        \
Parameter (x)  Constant (2)
```

## Démonstration concrète

```csharp
Expression<Func<int, bool>> expr = x => x > 5 && x < 100;

// On peut "visiter" l'arbre
var body = (BinaryExpression)expr.Body;          // &&
var left = (BinaryExpression)body.Left;          // x > 5
var right = (BinaryExpression)body.Right;        // x < 100

Console.WriteLine(body.NodeType);   // AndAlso
Console.WriteLine(left.NodeType);   // GreaterThan
Console.WriteLine(right.NodeType);  // LessThan

// Et on peut aussi le compiler en delegate pour l'exécuter
Func<int, bool> func = expr.Compile();
Console.WriteLine(func(42));  // true
Console.WriteLine(func(3));   // false
```

# Comportement côté exécution

## La même requête, deux exécutions différentes

```csharp
// Source IEnumerable<T> (liste en mémoire)
List<Client> clientsEnMemoire = GetClients();

var actifs1 = clientsEnMemoire
    .Where(c => c.IsActif)           // Func<Client, bool> → filtre en mémoire
    .OrderBy(c => c.Nom)             // tri en mémoire
    .Select(c => c.Nom)              // projection en mémoire
    .ToList();

// Source IQueryable<T> (DbSet Entity Framework)
IQueryable<Client> clientsDb = dbContext.Clients;

var actifs2 = clientsDb
    .Where(c => c.IsActif)           // Expression → traduit en WHERE IsActif = 1
    .OrderBy(c => c.Nom)             // Expression → traduit en ORDER BY Nom
    .Select(c => c.Nom)              // Expression → traduit en SELECT Nom
    .ToList();                        // exécute la requête SQL
```

Pour `clientsEnMemoire` :
1. **Toutes les données** sont déjà en mémoire.
2. `Where` itère sur chaque élément et appelle le delegate.
3. `OrderBy` charge les résultats filtrés dans un buffer et trie.
4. `Select` projette chaque élément trié.

Pour `clientsDb` :
1. Chaque opérateur **enrichit un arbre d'expression**.
2. Rien ne s'exécute tant qu'on n'appelle pas `ToList()`.
3. Au moment du `ToList()`, EF traduit tout l'arbre en **une seule requête SQL** :

```sql
SELECT [c].[Nom]
FROM [Clients] AS [c]
WHERE [c].[IsActif] = 1
ORDER BY [c].[Nom]
```

# Le piège de la bascule implicite

## `AsEnumerable()` et `ToList()` changent le monde d'exécution

```csharp
var resultat = dbContext.Clients
    .Where(c => c.IsActif)        // ← IQueryable : traduit en SQL
    .AsEnumerable()                // ← bascule vers IEnumerable !
    .Where(c => c.Nom.Length > 5) // ← IEnumerable : exécuté en mémoire
    .ToList();
```

La requête SQL générée est :

```sql
SELECT * FROM Clients WHERE IsActif = 1
-- Le filtre sur Nom.Length est fait EN MÉMOIRE après chargement
```

Le second `Where` n'est **pas** dans le SQL. Toutes les lignes actives sont chargées, puis filtrées en C#.

## Comment détecter sur quel "monde" on est

```csharp
IQueryable<Client> query = dbContext.Clients.Where(c => c.IsActif);
Console.WriteLine(query is IQueryable);    // True → monde SQL

IEnumerable<Client> enumerable = query.AsEnumerable();
// enumerable est toujours un IQueryable sous le capot,
// mais les opérateurs LINQ utilisés seront ceux de Enumerable (pas Queryable)
```

La règle de résolution des méthodes d'extension en C# fait que :
- Sur une variable typée `IQueryable<T>` → les méthodes de `Queryable` sont choisies (arbres d'expressions).
- Sur une variable typée `IEnumerable<T>` → les méthodes de `Enumerable` sont choisies (delegates).

# Ce que `IQueryable` ne peut PAS faire

## Méthodes C# non traduisibles

Un provider `IQueryable` ne peut traduire que les expressions qu'il connaît. Les méthodes C# arbitraires ne sont pas traduisibles.

```csharp
// ❌ EF ne sait pas traduire une méthode perso
var resultat = dbContext.Clients
    .Where(c => MonFormateur.Formater(c.Nom) == "ALICE")
    .ToList();
// → Exception : "The LINQ expression could not be translated"

// ❌ Certaines méthodes .NET ne sont pas supportées non plus
var resultat = dbContext.Clients
    .Where(c => c.DateCreation.ToString("yyyy-MM-dd") == "2025-01-01")
    .ToList();
// → Exception ou évaluation côté client selon la version d'EF
```

### Méthodes couramment supportées par EF Core

| Catégorie | Supporté | Non supporté |
|---|---|---|
| Comparaisons | `==`, `!=`, `>`, `<`, `>=`, `<=` | |
| String | `Contains`, `StartsWith`, `EndsWith`, `ToUpper`, `ToLower`, `Trim`, `Length` | `Format`, `Split`, regex |
| Math | `Math.Abs`, `Math.Round`, `Math.Floor` | `Math.Log`, `Math.Pow` (variable) |
| Nullable | `HasValue`, `Value`, `GetValueOrDefault` | |
| Collections | `Contains` (traduit en `IN`) | `Intersect`, `Union` sur des listes C# |
| Date | `DateTime.Now`, `.Year`, `.Month`, `.Day`, `.AddDays` | Formatage arbitraire |
| EF Functions | `EF.Functions.Like`, `EF.Functions.DateDiffDay` | |

## Pas de code impératif dans les expressions

Les arbres d'expressions ne supportent pas les instructions (if/else, boucles, try/catch, assignations…). Uniquement des **expressions**.

```csharp
// ❌ Pas possible dans un arbre d'expression
Expression<Func<Client, string>> expr = c =>
{
    if (c.IsVip) return "VIP";  // instruction → erreur de compilation
    return "Standard";
};

// ✅ Utiliser l'opérateur ternaire (c'est une expression)
Expression<Func<Client, string>> expr = c =>
    c.IsVip ? "VIP" : "Standard";

// EF traduit en : CASE WHEN IsVip = 1 THEN 'VIP' ELSE 'Standard' END
```

# Composition de requêtes

## `IQueryable` permet la composition dynamique

Un des grands avantages de `IQueryable<T>` est qu'on peut **composer** la requête progressivement sans l'exécuter :

```csharp
IQueryable<Commande> query = dbContext.Commandes;

// Ajout conditionnel de filtres
if (dateDebut.HasValue)
    query = query.Where(c => c.Date >= dateDebut.Value);

if (dateFin.HasValue)
    query = query.Where(c => c.Date <= dateFin.Value);

if (!string.IsNullOrEmpty(recherche))
    query = query.Where(c => c.Reference.Contains(recherche));

// Le SQL final ne contient QUE les filtres ajoutés
var resultats = await query
    .OrderByDescending(c => c.Date)
    .Take(50)
    .ToListAsync();
```

Si seul `dateDebut` est renseigné, le SQL sera :

```sql
SELECT TOP(50) * FROM Commandes WHERE Date >= @p0 ORDER BY Date DESC
```

## `IEnumerable` aussi, mais tout est en mémoire

On peut faire pareil avec `IEnumerable<T>`, mais toute la source doit déjà être en mémoire :

```csharp
IEnumerable<Commande> query = commandesEnMemoire;

if (dateDebut.HasValue)
    query = query.Where(c => c.Date >= dateDebut.Value);

// Fonctionne, mais filtre en mémoire sur des données déjà chargées
var resultats = query.ToList();
```

# Conversions entre les deux mondes

| Conversion | Méthode | Effet |
|---|---|---|
| `IQueryable` → `IEnumerable` | `.AsEnumerable()` | Bascule la résolution des opérateurs, **ne matérialise pas** |
| `IQueryable` → `List<T>` | `.ToList()` | Exécute la requête SQL et charge en mémoire |
| `IEnumerable` → `IQueryable` | `.AsQueryable()` | Wrapping, **ne recrée pas** un vrai provider SQL |

### Attention à `AsQueryable()`

```csharp
// ❌ AsQueryable() ne transforme PAS une liste en requête SQL
var listeEnMemoire = new List<int> { 1, 2, 3, 4, 5 };
var queryable = listeEnMemoire.AsQueryable();

// queryable est un IQueryable, mais le "provider" est EnumerableQuery
// → le Where est toujours exécuté en mémoire !
var resultat = queryable.Where(x => x > 3).ToList();
```

`AsQueryable()` est utile pour le **testing** (mocker un `IQueryable`) ou pour écrire du code générique, mais il ne crée pas de vraie traduction SQL.

# Quand utiliser quoi ?

| Scénario | Interface | Raison |
|---|---|---|
| Données déjà en mémoire (listes, tableaux) | `IEnumerable<T>` | Pas de traduction à faire |
| Requêter une base de données (EF) | `IQueryable<T>` | Traduction SQL, filtrage côté serveur |
| API publique d'un repository | `IQueryable<T>` ou `IEnumerable<T>` | Voir ci-dessous |
| Méthode utilitaire qui manipule des séquences | `IEnumerable<T>` | Plus générique, fonctionne partout |
| Tests unitaires | `IEnumerable<T>` ou `List<T>` | Pas besoin de provider |

### Le débat `IQueryable` dans les repositories

Exposer `IQueryable<T>` dans un repository donne de la flexibilité aux appelants (ils peuvent composer des filtres), mais **couple** le code appelant à la sémantique de la base de données et rend les tests plus complexes.

```csharp
// Option 1 : exposer IQueryable (flexible mais couplé)
public interface IClientRepository
{
    IQueryable<Client> GetAll(); // l'appelant ajoute ses filtres
}

// Option 2 : exposer IEnumerable avec des paramètres explicites (découplé)
public interface IClientRepository
{
    Task<IReadOnlyList<Client>> GetActifs(string? recherche = null, int? limit = null);
}
```

Il n'y a pas de réponse universelle. L'option 2 est préférable dans une **Clean Architecture** où on veut isoler la couche domaine de l'infrastructure.

# Résumé

| Aspect | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| **Namespace** | `System.Linq.Enumerable` | `System.Linq.Queryable` |
| **Paramètre du filtre** | `Func<T, bool>` (delegate) | `Expression<Func<T, bool>>` (arbre) |
| **Exécution** | En mémoire (CLR) | Traduite par un provider (SQL, etc.) |
| **Ce qui est envoyé** | Code IL compilé | Arbre d'expression analysable |
| **Composition** | Chaîne d'itérateurs en mémoire | Construction incrémentale d'un arbre |
| **Méthodes C# perso** | ✅ Toutes supportées | ❌ Seulement celles traduisibles |
| **Performance gros volumes** | Tout doit être en mémoire | Filtrage côté serveur |
| **Bascule** | `AsEnumerable()` depuis IQueryable | `AsQueryable()` (wrapping seul) |

# Pour aller plus loin : Entity Framework Core

Cet article parle beaucoup d'`IQueryable<T>` et de traduction SQL. Pour approfondir le sujet côté Entity Framework Core, consultez la série dédiée :

[Comprendre la "magie" derrière Entity Framework]({% post_url 2025-06-01-EntityPart1 %})

[Bonnes pratiques pour les requêtes (EF Core)]({% post_url 2025-07-14-EntityPart4 %}) 


