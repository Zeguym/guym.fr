---
title: "LINQ en C# : principes et fonctionnement"
tags: dotnet basics linq csharp
series: "Linq"
---

LINQ (Language Integrated Query) est l'une des fonctionnalités les plus puissantes de C#. Il permet d'interroger et de transformer des collections, des bases de données, du XML ou tout autre source de données avec une syntaxe unifiée, directement intégrée au langage. Comprendre ses principes — exécution différée, composition d'opérateurs, distinction `IEnumerable<T>` / `IQueryable<T>` — est indispensable pour écrire du code .NET lisible, maintenable et performant.

<!--more-->
{% include series.html %}
# Qu'est-ce que LINQ ?

**LINQ** (Language Integrated Query) est un ensemble de fonctionnalités introduites dans **C# 3.0** (.NET Framework 3.5, 2007) qui permet d'écrire des **requêtes sur des données** directement dans le langage C#, avec un typage fort et l'aide de l'IntelliSense.

Avant LINQ, interroger une base de données passait par des chaînes SQL en dur, interroger du XML par des chemins XPath, et parcourir des collections par des boucles `foreach` manuelles. LINQ unifie tout cela.

```csharp
// Avant LINQ
var resultats = new List<string>();
foreach (var p in personnes)
{
    if (p.Age >= 18)
        resultats.Add(p.Nom);
}
resultats.Sort();

// Avec LINQ
var resultats = personnes
    .Where(p => p.Age >= 18)
    .OrderBy(p => p.Nom)
    .Select(p => p.Nom)
    .ToList();
```

L'avantage est immédiat : le code est **déclaratif** (on décrit *ce qu'on veut*) plutôt qu'**impératif** (on décrit *comment le faire*).

# Les deux syntaxes

LINQ offre deux façons d'écrire les requêtes. Elles sont **fonctionnellement équivalentes** ; le compilateur transforme la syntaxe de requête en appels de méthodes.

## Syntaxe de requête (query syntax)

Elle ressemble à du SQL, mais attention : elle commence par `from` et finit par `select` (ou `group`).

```csharp
var adultes = from p in personnes
              where p.Age >= 18
              orderby p.Nom
              select p.Nom;
```

## Syntaxe de méthodes (method syntax / fluent syntax)

Elle utilise des **méthodes d'extension** chaînées, combinées avec des **expressions lambda**.

```csharp
var adultes = personnes
    .Where(p => p.Age >= 18)
    .OrderBy(p => p.Nom)
    .Select(p => p.Nom);
```

### Quelle syntaxe choisir ?

| Critère | Query syntax | Method syntax |
|---|---|---|
| Lisibilité pour les jointures | ✅ Plus lisible | Parfois verbeux |
| Accès à tous les opérateurs | ❌ Pas tous disponibles | ✅ Tous les opérateurs |
| Cohérence avec le reste du code | Variable | ✅ Style fluent classique |
| Convention la plus courante | Peu utilisée seule | ✅ Standard de facto |

En pratique, la **syntaxe de méthodes** est la plus utilisée. La syntaxe de requête reste utile pour les jointures complexes ou quand elle améliore la lisibilité.

# Les briques fondamentales

## Les expressions lambda

Les lambdas sont au cœur de LINQ. Ce sont des **fonctions anonymes** passées en paramètre aux opérateurs LINQ.

```csharp
// Lambda simple : un paramètre, une expression
Func<int, bool> estPair = x => x % 2 == 0;

// Lambda avec corps
Func<int, string> decrire = x =>
{
    if (x > 0) return "positif";
    if (x < 0) return "négatif";
    return "zéro";
};
```

## Les méthodes d'extension

Les opérateurs LINQ (`Where`, `Select`, `OrderBy`…) sont des **méthodes d'extension** définies dans la classe `System.Linq.Enumerable` (pour `IEnumerable<T>`) et `System.Linq.Queryable` (pour `IQueryable<T>`).

```csharp
// Ce qu'on écrit :
personnes.Where(p => p.Age >= 18);

// Ce que le compilateur voit :
Enumerable.Where(personnes, p => p.Age >= 18);
```

C'est pour cela qu'il faut toujours avoir le `using System.Linq;` (implicite depuis .NET 6 avec les *global usings*).

## Les types anonymes

LINQ utilise souvent des **types anonymes** pour projeter les données :

```csharp
var resultats = personnes
    .Select(p => new { p.Nom, p.Age })
    .ToList();

// Chaque élément a des propriétés Nom et Age, 
// typées automatiquement par le compilateur
```

# `IEnumerable<T>` vs `IQueryable<T>`

C'est **la distinction la plus importante** à comprendre pour bien utiliser LINQ.

## `IEnumerable<T>` — LINQ to Objects

- Travaille **en mémoire**, sur des collections déjà chargées.
- Les lambdas sont compilées en **delegates** (`Func<T, bool>`).
- Chaque opérateur itère sur les éléments un par un.

```csharp
List<Personne> personnes = GetPersonnes(); // données en mémoire

var adultes = personnes
    .Where(p => p.Age >= 18)   // filtre en mémoire
    .ToList();
```

## `IQueryable<T>` — LINQ to Entities (et autres providers)

- Travaille avec un **provider** externe (Entity Framework, base de données…).
- Les lambdas sont compilées en **arbres d'expressions** (`Expression<Func<T, bool>>`).
- La requête est **traduite** (par exemple en SQL) et exécutée **côté serveur**.

```csharp
IQueryable<Personne> personnes = dbContext.Personnes; // DbSet<T>

var adultes = personnes
    .Where(p => p.Age >= 18)   // traduit en SQL : WHERE Age >= 18
    .ToList();                  // exécution SQL à ce moment-là
```

### Pourquoi c'est crucial ?

| Aspect | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| Exécution | En mémoire (client) | Côté serveur (base de données) |
| Filtre | Après avoir tout chargé | Avant chargement (SQL WHERE) |
| Performance | Peut être coûteux sur gros volumes | Optimisé par le provider |
| Expression | `Func<T, bool>` (delegate) | `Expression<Func<T, bool>>` (arbre) |

**Piège classique** : appeler `.AsEnumerable()` ou `.ToList()` trop tôt force le chargement de toutes les données en mémoire, puis les filtres suivants s'exécutent côté client.

```csharp
// ❌ Mauvais : charge TOUT puis filtre en mémoire
var resultat = dbContext.Commandes
    .ToList()                          // SELECT * FROM Commandes
    .Where(c => c.Montant > 1000);     // filtre en mémoire

// ✅ Bon : filtre côté SQL
var resultat = dbContext.Commandes
    .Where(c => c.Montant > 1000)      // WHERE Montant > 1000
    .ToList();                          // exécution SQL filtrée
```

# L'exécution différée (deferred execution)

C'est le concept **le plus important** de LINQ.

## Principe

Quand on écrit une requête LINQ, **rien ne s'exécute immédiatement**. On construit un **pipeline de transformations**. L'exécution ne se déclenche que lorsqu'on **consomme** les résultats.

```csharp
var query = nombres.Where(n => n > 5); // rien ne se passe encore

foreach (var n in query)  // l'exécution se déclenche ICI
{
    Console.WriteLine(n);
}
```

## Ce qui déclenche l'exécution

Les opérateurs LINQ se divisent en deux catégories :

### Opérateurs différés (ne déclenchent PAS l'exécution)

Ils retournent un `IEnumerable<T>` ou `IQueryable<T>` et ne font que **composer** le pipeline :

- `Where`, `Select`, `SelectMany`
- `OrderBy`, `ThenBy`, `OrderByDescending`
- `Skip`, `Take`
- `Distinct`, `GroupBy`, `Join`
- `Concat`, `Union`, `Intersect`, `Except`

### Opérateurs immédiats (DÉCLENCHENT l'exécution)

Ils consomment la séquence et retournent un résultat concret :

- **Matérialisation** : `ToList()`, `ToArray()`, `ToDictionary()`, `ToHashSet()`
- **Élément unique** : `First()`, `FirstOrDefault()`, `Single()`, `Last()`, `ElementAt()`
- **Agrégation** : `Count()`, `Sum()`, `Average()`, `Min()`, `Max()`, `Aggregate()`
- **Test** : `Any()`, `All()`, `Contains()`
- **Itération** : `foreach`

## Conséquences pratiques

### La source peut changer entre la définition et l'exécution

```csharp
var liste = new List<int> { 1, 2, 3 };
var query = liste.Where(n => n > 1);

liste.Add(4); // on modifie la source AVANT d'itérer

var resultat = query.ToList(); // [2, 3, 4] — le 4 est inclus !
```

### Chaque itération ré-exécute la requête

```csharp
var query = personnes.Where(p => p.Age >= 18);

// Deux itérations = deux exécutions du filtre
var count = query.Count();       // 1ère exécution
var list  = query.ToList();      // 2ème exécution
```

Si c'est un `IQueryable`, cela signifie **deux requêtes SQL** envoyées à la base. Pour éviter cela, matérialisez une seule fois avec `ToList()` :

```csharp
var adultes = personnes.Where(p => p.Age >= 18).ToList(); // 1 seule exécution
var count = adultes.Count;   // propriété List<T>, pas de ré-exécution
```

# Les opérateurs LINQ essentiels

## Filtrage

```csharp
// Where : filtre selon un prédicat
var majeurs = personnes.Where(p => p.Age >= 18);

// OfType : filtre par type
var chaines = objets.OfType<string>();
```

## Projection

```csharp
// Select : transforme chaque élément
var noms = personnes.Select(p => p.Nom);

// SelectMany : "aplatit" les collections imbriquées
var tousLesEmails = clients.SelectMany(c => c.Emails);
// Si chaque client a plusieurs emails, on obtient une seule séquence plate
```

## Tri

```csharp
var triParNom = personnes
    .OrderBy(p => p.Nom)
    .ThenByDescending(p => p.Age);
```

## Agrégation

```csharp
var total       = commandes.Sum(c => c.Montant);
var moyenne     = commandes.Average(c => c.Montant);
var nbAdultes   = personnes.Count(p => p.Age >= 18);
var plusVieux    = personnes.Max(p => p.Age);
```

## Regroupement

```csharp
var parVille = personnes
    .GroupBy(p => p.Ville)
    .Select(g => new 
    { 
        Ville = g.Key, 
        Nombre = g.Count() 
    });
```

## Jointures

```csharp
// Join : jointure interne (comme INNER JOIN en SQL)
var commandes = clients
    .Join(commandes,
        client => client.Id,
        commande => commande.ClientId,
        (client, commande) => new { client.Nom, commande.Montant });

// GroupJoin : jointure groupée (comme LEFT JOIN avec regroupement)
var clientsAvecCommandes = clients
    .GroupJoin(commandes,
        c => c.Id,
        cmd => cmd.ClientId,
        (client, cmds) => new { client.Nom, Commandes = cmds.ToList() });
```

En syntaxe de requête, les jointures sont plus lisibles :

```csharp
var resultat = from c in clients
               join cmd in commandes on c.Id equals cmd.ClientId
               select new { c.Nom, cmd.Montant };
```

## Partitionnement

```csharp
var page = personnes
    .OrderBy(p => p.Nom)
    .Skip(20)    // sauter les 20 premiers
    .Take(10);   // prendre les 10 suivants (page 3 de taille 10)
```

## Ensembles

```csharp
var union     = liste1.Union(liste2);        // éléments des deux, sans doublons
var inter     = liste1.Intersect(liste2);    // éléments communs
var diff      = liste1.Except(liste2);       // dans liste1 mais pas dans liste2
var distinct  = liste1.Distinct();           // supprime les doublons
```

## Vérification d'existence

```csharp
bool auMoinsUnAdulte = personnes.Any(p => p.Age >= 18);
bool tousMajeurs     = personnes.All(p => p.Age >= 18);
bool contientAlice   = personnes.Any(p => p.Nom == "Alice");
```

# Créer sa propre méthode LINQ

LINQ est extensible. On peut écrire ses propres opérateurs sous forme de méthodes d'extension sur `IEnumerable<T>` :

```csharp
public static class EnumerableExtensions
{
    /// <summary>
    /// Filtre les éléments non null d'une séquence.
    /// </summary>
    public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T?> source)
        where T : class
    {
        foreach (var item in source)
        {
            if (item is not null)
                yield return item;
        }
    }
}

// Utilisation
var noms = personnes
    .Select(p => p.Nom)
    .WhereNotNull()
    .ToList();
```

Le mot-clé `yield return` génère un **itérateur** qui respecte naturellement l'exécution différée.

# Bonnes pratiques et pièges courants

## 1. Ne pas mélanger `IQueryable` et opérations non traduisibles

```csharp
// ❌ Erreur à l'exécution : EF ne sait pas traduire MaMethode() en SQL
var resultat = dbContext.Personnes
    .Where(p => MaMethodeCustom(p.Nom))
    .ToList();

// ✅ Charger d'abord, puis filtrer en mémoire
var resultat = dbContext.Personnes
    .AsEnumerable()              // bascule en IEnumerable
    .Where(p => MaMethodeCustom(p.Nom))
    .ToList();
```

## 2. Matérialiser quand on itère plusieurs fois

```csharp
// ❌ Deux passages sur la requête (deux requêtes SQL si IQueryable)
var query = GetData().Where(x => x.IsActive);
var count = query.Count();
var items = query.ToList();

// ✅ Un seul passage
var items = GetData().Where(x => x.IsActive).ToList();
var count = items.Count;
```

## 3. Préférer `Any()` à `Count() > 0`

```csharp
// ❌ Count() parcourt toute la collection
if (personnes.Count() > 0) { ... }

// ✅ Any() s'arrête au premier élément trouvé
if (personnes.Any()) { ... }
```

## 4. Attention à la capture de variables dans les closures

```csharp
var filtres = new List<Func<int, bool>>();
for (int i = 0; i < 5; i++)
{
    filtres.Add(x => x == i); // ⚠️ capture de la variable i, pas de la valeur
}
// Tous les filtres testent x == 5 (valeur finale de i)

// ✅ Capturer une copie locale
for (int i = 0; i < 5; i++)
{
    var copie = i;
    filtres.Add(x => x == copie);
}
```

## 5. Utiliser les surcharges avec index quand c'est utile

```csharp
var indexés = noms
    .Select((nom, index) => $"{index + 1}. {nom}")
    .ToList();
// ["1. Alice", "2. Bob", "3. Charlie"]
```

# Résumé

| Concept | À retenir |
|---|---|
| **LINQ** | Requêtes intégrées au langage, typées, composables |
| **Deux syntaxes** | Query (`from ... select`) et Method (`.Where().Select()`) — équivalentes |
| **Exécution différée** | La requête ne s'exécute que quand on consomme les résultats |
| **`IEnumerable<T>`** | Exécution en mémoire (LINQ to Objects) |
| **`IQueryable<T>`** | Traduction en requête externe (SQL via EF, etc.) |
| **Opérateurs immédiats** | `ToList()`, `Count()`, `First()`, `Any()`… déclenchent l'exécution |
| **Extensibilité** | On peut créer ses propres opérateurs avec des méthodes d'extension |
| **Performance** | Filtrer avant de matérialiser, `Any()` plutôt que `Count() > 0`, matérialiser une seule fois |
