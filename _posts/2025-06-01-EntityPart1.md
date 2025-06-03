---
title: Comprendre la "magie" derrière Entity Framework (EF Core)
tags: dotnet efcore 
series: "Entity Framework"
---

Dans cette première partie consacrée à Entity Framework, l’objectif est de lever le voile sur la “magie” qui se cache derrière les requêtes LINQ et le fonctionnement interne d’EF Core. Nous allons voir ce qu’est réellement une expression, comment les lambdas sont transformées en arbres d’expressions puis traduites en SQL, et en quoi cela diffère d’une simple fonction exécutée en mémoire. Nous aborderons aussi les différences fondamentales entre IQueryable et IEnumerable, ainsi que la notion de matérialisation des requêtes, afin de mieux comprendre quand et où s’exécute réellement le code (côté serveur ou côté client).

<!--more-->
 {% include series.html %}


# 1. Qu’est-ce qu’une expression ?

Une **expression** est généralement une lambda (ex : `x => x.Prop == value`) qui décrit une opération à effectuer sur les données.  
Avec Entity Framework, ces expressions sont traduites en requêtes SQL exécutées sur la base de données.

En C#, une **expression** est un fragment de code qui produit une valeur. Dans le contexte de LINQ et Entity Framework, le terme **expression** désigne souvent une **expression lambda** ou un **arbre d'expression** (`Expression<TDelegate>`).

## 1. Expression lambda

C’est une fonction anonyme, souvent utilisée pour décrire des opérations sur des collections. Exemple :

```csharp
x => x.Age > 18
```

Ici, `x => x.Age > 18` est une expression lambda qui prend un objet `x` et retourne `true` si son âge est supérieur à 18.

## 2. Arbre d’expression (`Expression<T>`)

Un **arbre d’expression** est une représentation objet d’une expression lambda.  
Entity Framework utilise ces arbres pour analyser et convertir le code C# en requête SQL.

Exemple :

```csharp
Expression<Func<User, bool>> expr = u => u.IsActive;
```

- Ici, `expr` n’est pas une fonction, mais une description de la logique à appliquer.
- Entity Framework lit cet arbre pour générer la requête SQL correspondante.

## 3. Différence avec une fonction

- Une **fonction** (`Func<T, bool>`) s’exécute en mémoire, c'est du code compilé qui s'exécute sur des objets déjà chargés.
- Une **expression** (`Expression<Func<T, bool>>`) décrit la logique (exprime une intention) qu'entity peut traduire en SQL et exécuter sur la base de données. Il est tout à fait possible de transformer une expression en fonction en utilisant l'extension `.Compile()`.

```csharp
Expression<Func<User, bool>> expr = u => u.IsActive;
Func<User, bool> func = expr.Compile();
func(user);
```

# 2. Qu’est-ce qu’une expression Linq ?

Une **expression Linq** est une syntaxe permettant de décrire une requête sur des collections d’objets, souvent sous forme de lambda.  
Avec Entity Framework, ces expressions sont traduites en requêtes SQL et exécutées sur la base de données.

## Exemple simple

```csharp
var clientsActifs = db.Clients.Where(c => c.IsActive).ToList();
```

- Ici, `c => c.IsActive` est une **expression lambda**.
- Entity Framework analyse cette expression et la convertit en SQL :

```sql
SELECT * FROM Clients WHERE IsActive = 1
```

## Pourquoi c’est important ?

- Les expressions Linq permettent d’écrire des requêtes de façon déclarative, directement dans le code C#.
- Entity Framework lit et interprète ces expressions pour générer la requête la plus efficace possible côté serveur (SQL). **Le code du Where tel que l'on connaît en Linq n'est pas exécuté**
- Il est donc important de connaître la liste des opérations traduisibles en requêtes. Entity propose un socle commun mais chaque SGBD propose un set de méthodes spécifiques (pour [postgres](https://www.npgsql.org/efcore/mapping/translations.html "https://www.npgsql.org/efcore/mapping/translations.html"), pour [sql server](https://learn.microsoft.com/en-us/ef/core/providers/sql-server/functions "https://learn.microsoft.com/en-us/ef/core/providers/sql-server/functions") )

## Types d’expressions courantes

- **Filtrage** : `.Where(x => ...)`
- **Projection** : `.Select(x => ...)`
- **Tri** : `.OrderBy(x => ...)`
- **Agrégation** : `.Any(x => ...)`, `.Count(x => ...)`, `.Sum(x => ...)`

## Avantage

- Le code C# est rédigé par le développeur, tandis qu’Entity Framework se charge de la traduction en requêtes SQL adaptées au SGBD cible.
- Le développeur bénéficie de l’analyse statique du compilateur ainsi que de la sécurité offerte par le typage fort.

# 3. **Expression Trees (`Expression<T>`)**

## Transformation du code en requête

Entity Framework utilise des **arbres d’expressions** (`Expression<T>`) pour analyser et convertir les requêtes LINQ en SQL.

**Exemple :**

```csharp
db.Products.Where(p => p.Price > 100);
```

Ici, la lambda `p => p.Price > 100` est convertie en un arbre d’expression, que EF Core analyse pour générer la requête SQL correspondante. Il n'est donc pas possible d'utiliser une fonction directement dans le code d'une requête entity

Le code suivant est donc invalide :

```csharp
db.Products.Where(p => isPriceValid(p));
bool isPriceValid(Product p) => p.Price> 100 ;
```

Dans le cadre d'un code clair, il est toujours possible de factoriser son code métier en écrivant des expressions et non plus des fonctions et de compiler l'expression pour l'exécuter de façon classique :

```csharp
Expression<Func<Product, bool>> isPriceValidExp = ( p) => p.Price> 100;
db.Products.Where(isPriceValidExp);

var produit =  new Product{ Price = 102};
isPriceValid = isPriceValidExp.Compile();

Console.WrileLine( isPriceValid(produit));
```

Il est important d'expliciter le type de l'expression car par défaut ce sera un type `Func` généré

```csharp
var isPriceValidExp = ( p) => p.Price> 100; // Var sera de type Func<Produit, bool>
```

## Différence entre `Func<T, bool>` et `Expression<Func<T, bool>>`

- `Func<T, bool>` : fonction exécutée en mémoire (côté client).
- `Expression<Func<T, bool>>` : description de la requête, traduite en SQL (côté serveur).

**Exemple :**

```csharp
// Traduit en SQL (côté serveur)
db.Products.Where(p => p.Price > 100) // Expression<Func<Product, bool>>/
// / Exécuté en mémoire (côté client)
db.Products.ToList().Where(p => p.Price > 100) // Func<Product, bool>
```

- La première version remonte au C# les produits dont le prix est supérieur à 100
- Pour la seconde version, le .ToList() brise la chaîne des expressions pour des fonctions. Le C# va recevoir l'intégralité des produits et puis les parcourir pour ne sélectionner que ceux dont le prix est supérieur à 100

**Bonne pratique :** Toujours filtrer avant `.ToList()` pour rester côté SQL.

En résumé :

- Les expressions permettent à Entity Framework d’optimiser les requêtes et de ne charger que les données nécessaires.
- Elles évitent de transférer trop de données en mémoire et améliorent les performances.

# `IQueryable` vs `IEnumerable`

Si l'on reprend l'exemple du chapitre précédent :

```csharp
db.Products.Where(p => p.Price > 100);
db.Products.ToList().Where(p => p.Price > 100);
```

le 'Where' a un comportement d'exécution fondamentalement différent :

- l'un va s’exécuter en tant que SQL sur le serveur de base de données
- l'autre va faire un C# foreach en mémoire

Car il existe deux surcharges :

```csharp
// System.Linq.Queryable
public static IQueryable<TSource> Where<TSource>(this IQueryable<TSource> source, Expression<Func<TSource, bool>> predicate);
```

Cette version enchaîne des expressions dans des IQueryable et est traduisible en SQL.

```csharp
// System.Linq
public static IEnumerable<TSource> Where<TSource>(this IEnumerable<TSource> source, Func<TSource, bool> predicate);
```

Cette version enchaîne des fonctions dans des IEnumerable et les exécute en mémoire.

La principale différence entre `IQueryable` et `IEnumerable` réside dans la façon dont ils traitent les données et où l’exécution de la requête a lieu :

- **IEnumerable** :

  - Représente une collection d’objets en mémoire.
  - Les opérations (filtrage, projection, etc.) sont exécutées côté client, après chargement des données.
  - Utilisé principalement pour manipuler des collections déjà chargées (ex : listes, tableaux).

- **IQueryable** :
  - Représente une requête sur une source de données distante (ex : base SQL).
  - Les opérations sont traduites en requête (ex : SQL) et exécutées côté serveur.
  - Permet d’optimiser le transfert de données en ne ramenant que ce qui est nécessaire.

Utilisez `IQueryable` pour construire des requêtes côté serveur (avant `.ToList()`), et `IEnumerable` pour manipuler des données déjà chargées en mémoire.

# Matérialisation des requêtes.

En Linq que ce soit avec des IQueryable ou des IEnumerable c'est l'itération qui déclenche l'exécution.

```csharp
var data = db.Products.Where(p => p.Price > 100); // ne fait rien

List<Product> products = .....
var data = products.Where(p => p.Price > 100); // ne fait rien

```

Dans l'exemple du chapitre précédent (`db.Products.ToList().Where(p => p.Price > 100);`) on a vu que le `.ToList()` déclenchait un select \*. `db.Products` la source de données est un IQueryable. Le .ToList() a le prototype suivant.

```csharp
public static List<TSource> ToList<TSource>(this IEnumerable<TSource> source)
```

Il ne travaille que sur des `Enumerable` cela est possible car `IQueryable` dérive de `IEnumerable`

```csharp
public interface IQueryable<out T> : IEnumerable<T>, IEnumerable, IQueryable
```

C'est l’accès au `GetEnumerator()` de `IEnumerable` implémenté dans le `IQueryable` qui va générer et exécuter la requête
