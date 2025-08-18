---
title: "Comprendre LINQ par l'implémentation : recoder les opérateurs"
tags: dotnet basics linq csharp implementation
series: "Linq"
---

LINQ semble magique quand on l'utilise, mais sous le capot, ce sont des **méthodes d'extension**, des **itérateurs** (`yield return`) et des **delegates** qui font tout le travail. La meilleure façon de comprendre LINQ en profondeur est de **recoder soi-même** ses opérateurs principaux. Cet article propose une implémentation simplifiée de LINQ to Objects, opérateur par opérateur.

<!--more-->
{% include series.html %}
# Les fondations : `IEnumerable<T>` et `yield return`

Toute l'implémentation de LINQ repose sur deux mécanismes du langage C# :

1. **`IEnumerable<T>`** : l'interface que toute séquence doit implémenter.
2. **`yield return`** : le mot-clé qui permet au compilateur de générer un **itérateur** (une machine à états) automatiquement.

## Rappel sur `IEnumerable<T>`

```csharp
public interface IEnumerable<out T> : IEnumerable
{
    IEnumerator<T> GetEnumerator();
}

public interface IEnumerator<out T> : IDisposable, IEnumerator
{
    T Current { get; }
    bool MoveNext();
    void Reset();
}
```

Quand on écrit `foreach (var item in collection)`, le compilateur appelle `GetEnumerator()` puis boucle sur `MoveNext()` / `Current`.

## Le rôle de `yield return`

Sans `yield return`, il faudrait écrire manuellement une classe qui implémente `IEnumerator<T>` avec une machine à états. C'est exactement ce que le compilateur fait pour nous :

```csharp
// Ce qu'on écrit
public static IEnumerable<int> Naturels(int max)
{
    for (int i = 0; i < max; i++)
        yield return i;
}

// Ce que le compilateur génère (simplifié)
private sealed class NaturelsIterator : IEnumerable<int>, IEnumerator<int>
{
    private int _state = 0;
    private int _current;
    private int _max;
    private int _i;

    public NaturelsIterator(int max) => _max = max;

    public int Current => _current;

    public bool MoveNext()
    {
        switch (_state)
        {
            case 0:
                _i = 0;
                _state = 1;
                goto case 1;
            case 1:
                if (_i < _max)
                {
                    _current = _i;
                    _i++;
                    return true;
                }
                _state = -1;
                return false;
            default:
                return false;
        }
    }

    // ... GetEnumerator, Dispose, Reset omis
}
```

Le `yield return` produit naturellement une **exécution différée** : le code du corps de la méthode ne s'exécute que quand l'appelant appelle `MoveNext()`.

# La classe de base : `MyEnumerable`

On va créer une classe statique `MyEnumerable` qui contiendra nos méthodes d'extension, exactement comme le fait `System.Linq.Enumerable` dans le framework.

```csharp
public static class MyEnumerable
{
    // Nos opérateurs iront ici
}
```

Chaque opérateur suit le même schéma :
1. C'est une **méthode d'extension** sur `IEnumerable<T>`.
2. Elle **valide les paramètres** immédiatement (pas de `yield` dans la méthode publique).
3. Elle délègue à une **méthode privée** qui utilise `yield return` pour l'exécution différée.

## Pourquoi séparer validation et itération ?

Parce que `yield return` rend l'exécution différée. Si on valide les paramètres dans la même méthode, l'exception ne sera levée que quand on itère, pas quand on appelle la méthode :

```csharp
// ❌ L'exception n'est levée qu'au foreach, pas à l'appel
public static IEnumerable<T> MauvaisWhere<T>(
    this IEnumerable<T> source, Func<T, bool> predicate)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    // avec yield, le code ci-dessus ne s'exécute qu'au MoveNext() !
    foreach (var item in source)
        if (predicate(item))
            yield return item;
}

// ✅ L'exception est levée immédiatement à l'appel
public static IEnumerable<T> BonWhere<T>(
    this IEnumerable<T> source, Func<T, bool> predicate)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    if (predicate == null) throw new ArgumentNullException(nameof(predicate));
    return BonWhereIterator(source, predicate);
}

private static IEnumerable<T> BonWhereIterator<T>(
    IEnumerable<T> source, Func<T, bool> predicate)
{
    foreach (var item in source)
        if (predicate(item))
            yield return item;
}
```

C'est exactement ce que fait l'implémentation officielle de .NET.

# Implémentation des opérateurs

## `Where` — Filtrage

Le plus simple des opérateurs. On parcourt la source et on ne retourne que les éléments qui satisfont le prédicat.

```csharp
public static IEnumerable<TSource> Where<TSource>(
    this IEnumerable<TSource> source,
    Func<TSource, bool> predicate)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    if (predicate == null) throw new ArgumentNullException(nameof(predicate));
    return WhereIterator(source, predicate);
}

private static IEnumerable<TSource> WhereIterator<TSource>(
    IEnumerable<TSource> source,
    Func<TSource, bool> predicate)
{
    foreach (var element in source)
    {
        if (predicate(element))
            yield return element;
    }
}
```

**Ce qu'il faut retenir** : grâce à `yield return`, chaque élément est produit **un par un**. Si l'appelant arrête d'itérer (par exemple avec `First()`), le reste de la source n'est jamais parcouru.

## `Select` — Projection

Transforme chaque élément via une fonction de projection.

```csharp
public static IEnumerable<TResult> Select<TSource, TResult>(
    this IEnumerable<TSource> source,
    Func<TSource, TResult> selector)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    if (selector == null) throw new ArgumentNullException(nameof(selector));
    return SelectIterator(source, selector);
}

private static IEnumerable<TResult> SelectIterator<TSource, TResult>(
    IEnumerable<TSource> source,
    Func<TSource, TResult> selector)
{
    foreach (var element in source)
    {
        yield return selector(element);
    }
}
```

## `SelectMany` — Aplatissement

C'est le `flatMap` de LINQ. Chaque élément produit une sous-séquence, et on aplatit le tout.

```csharp
public static IEnumerable<TResult> SelectMany<TSource, TResult>(
    this IEnumerable<TSource> source,
    Func<TSource, IEnumerable<TResult>> selector)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    if (selector == null) throw new ArgumentNullException(nameof(selector));
    return SelectManyIterator(source, selector);
}

private static IEnumerable<TResult> SelectManyIterator<TSource, TResult>(
    IEnumerable<TSource> source,
    Func<TSource, IEnumerable<TResult>> selector)
{
    foreach (var element in source)
    {
        foreach (var subElement in selector(element))
        {
            yield return subElement;
        }
    }
}
```

## `Any` — Existence (opérateur immédiat)

Premier exemple d'opérateur **immédiat** : il consomme la séquence et retourne un résultat tout de suite. Pas de `yield return` ici.

```csharp
public static bool Any<TSource>(
    this IEnumerable<TSource> source,
    Func<TSource, bool> predicate)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    if (predicate == null) throw new ArgumentNullException(nameof(predicate));

    foreach (var element in source)
    {
        if (predicate(element))
            return true;  // court-circuit : on s'arrête au premier match
    }
    return false;
}

// Surcharge sans prédicat : vérifie si la séquence contient au moins un élément
public static bool Any<TSource>(this IEnumerable<TSource> source)
{
    if (source == null) throw new ArgumentNullException(nameof(source));

    using var enumerator = source.GetEnumerator();
    return enumerator.MoveNext();
}
```

## `First` et `FirstOrDefault` — Accès au premier élément

```csharp
public static TSource First<TSource>(
    this IEnumerable<TSource> source,
    Func<TSource, bool> predicate)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    if (predicate == null) throw new ArgumentNullException(nameof(predicate));

    foreach (var element in source)
    {
        if (predicate(element))
            return element;
    }

    throw new InvalidOperationException("La séquence ne contient aucun élément correspondant.");
}

public static TSource? FirstOrDefault<TSource>(
    this IEnumerable<TSource> source,
    Func<TSource, bool> predicate)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    if (predicate == null) throw new ArgumentNullException(nameof(predicate));

    foreach (var element in source)
    {
        if (predicate(element))
            return element;
    }

    return default;
}
```

## `Count` — Comptage

```csharp
public static int Count<TSource>(this IEnumerable<TSource> source)
{
    if (source == null) throw new ArgumentNullException(nameof(source));

    // Optimisation : si c'est déjà une collection, on connaît la taille
    if (source is ICollection<TSource> collection)
        return collection.Count;

    int count = 0;
    using var enumerator = source.GetEnumerator();
    while (enumerator.MoveNext())
        count++;

    return count;
}

public static int Count<TSource>(
    this IEnumerable<TSource> source,
    Func<TSource, bool> predicate)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    if (predicate == null) throw new ArgumentNullException(nameof(predicate));

    int count = 0;
    foreach (var element in source)
    {
        if (predicate(element))
            count++;
    }
    return count;
}
```

**Point intéressant** : l'optimisation `ICollection<T>` est un pattern récurrent dans l'implémentation officielle. LINQ vérifie souvent si la source implémente une interface plus spécifique pour court-circuiter l'itération. Par expemple, `Count` peut retourner immédiatement la taille d'une liste ou d'un tableau sans parcourir les éléments :

```csharp
public static int Count<TSource>(this IEnumerable<TSource> source)
{
    if (source == null) throw new ArgumentNullException(nameof(source));

    // Optimisation : si c'est une collection, accès direct à la propriété Count
    if (source is ICollection<TSource> collection)
        return collection.Count;

    // Sinon, on itère et on compte
    int count = 0;
    using var enumerator = source.GetEnumerator();
    while (enumerator.MoveNext())
        count++;

    return count;
}
```


## `ToList` — Matérialisation

```csharp
public static List<TSource> ToList<TSource>(this IEnumerable<TSource> source)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    return new List<TSource>(source);
}
```

C'est tout. Le constructeur de `List<T>` fait déjà le travail d'itération et de copie.

## `OrderBy` — Tri (opérateur bufferisé)

Le tri est un cas spécial : c'est un opérateur **différé** mais **bufferisé**. Il doit lire toute la source avant de produire le premier résultat.

```csharp
public static IOrderedEnumerable<TSource> OrderBy<TSource, TKey>(
    this IEnumerable<TSource> source,
    Func<TSource, TKey> keySelector)
    where TKey : IComparable<TKey>
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    if (keySelector == null) throw new ArgumentNullException(nameof(keySelector));
    return new OrderedEnumerable<TSource, TKey>(source, keySelector, descending: false);
}

// Implémentation simplifiée de IOrderedEnumerable
public class OrderedEnumerable<TSource, TKey> : IOrderedEnumerable<TSource>
    where TKey : IComparable<TKey>
{
    private readonly IEnumerable<TSource> _source;
    private readonly Func<TSource, TKey> _keySelector;
    private readonly bool _descending;

    public OrderedEnumerable(
        IEnumerable<TSource> source,
        Func<TSource, TKey> keySelector,
        bool descending)
    {
        _source = source;
        _keySelector = keySelector;
        _descending = descending;
    }

    public IEnumerator<TSource> GetEnumerator()
    {
        // Bufferisation : on charge tout en mémoire pour trier
        var buffer = _source.ToList();
        
        buffer.Sort((a, b) =>
        {
            int cmp = _keySelector(a).CompareTo(_keySelector(b));
            return _descending ? -cmp : cmp;
        });

        foreach (var element in buffer)
            yield return element;
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

    // Nécessaire pour supporter ThenBy
    public IOrderedEnumerable<TSource> CreateOrderedEnumerable<TNextKey>(
        Func<TSource, TNextKey> keySelector,
        IComparer<TNextKey>? comparer,
        bool descending)
    {
        throw new NotImplementedException("Simplifié pour l'article");
    }
}
```

**Point clé** : même si `OrderBy` est différé (le tri ne se fait pas à l'appel), il est **bufferisé** — quand on commence à itérer, toute la source doit être chargée en mémoire pour pouvoir trier. C'est une différence importante avec `Where` ou `Select` qui sont de vrais opérateurs *streaming*.

## `Skip` et `Take` — Partitionnement

```csharp
public static IEnumerable<TSource> Skip<TSource>(
    this IEnumerable<TSource> source, int count)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    return SkipIterator(source, count);
}

private static IEnumerable<TSource> SkipIterator<TSource>(
    IEnumerable<TSource> source, int count)
{
    int skipped = 0;
    foreach (var element in source)
    {
        if (skipped < count)
        {
            skipped++;
            continue;
        }
        yield return element;
    }
}

public static IEnumerable<TSource> Take<TSource>(
    this IEnumerable<TSource> source, int count)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    return TakeIterator(source, count);
}

private static IEnumerable<TSource> TakeIterator<TSource>(
    IEnumerable<TSource> source, int count)
{
    int taken = 0;
    foreach (var element in source)
    {
        if (taken >= count)
            yield break;  // on arrête complètement l'itération

        yield return element;
        taken++;
    }
}
```

**`yield break`** est l'équivalent de `return` dans un itérateur : il termine la séquence.

## `Distinct` — Suppression des doublons

```csharp
public static IEnumerable<TSource> Distinct<TSource>(
    this IEnumerable<TSource> source)
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    return DistinctIterator(source);
}

private static IEnumerable<TSource> DistinctIterator<TSource>(
    IEnumerable<TSource> source)
{
    var seen = new HashSet<TSource>();
    foreach (var element in source)
    {
        if (seen.Add(element))  // Add retourne false si déjà présent
            yield return element;
    }
}
```

Cet opérateur est **semi-bufferisé** : il maintient un `HashSet` en mémoire, mais produit les éléments en streaming (dès qu'un élément nouveau est rencontré, il est émis).

## `GroupBy` — Regroupement

```csharp
public static IEnumerable<IGrouping<TKey, TSource>> GroupBy<TSource, TKey>(
    this IEnumerable<TSource> source,
    Func<TSource, TKey> keySelector)
    where TKey : notnull
{
    if (source == null) throw new ArgumentNullException(nameof(source));
    if (keySelector == null) throw new ArgumentNullException(nameof(keySelector));
    return GroupByIterator(source, keySelector);
}

private static IEnumerable<IGrouping<TKey, TSource>> GroupByIterator<TSource, TKey>(
    IEnumerable<TSource> source,
    Func<TSource, TKey> keySelector)
    where TKey : notnull
{
    // Bufferisation : on doit tout lire pour connaître tous les groupes
    var groups = new Dictionary<TKey, List<TSource>>();
    var orderedKeys = new List<TKey>();

    foreach (var element in source)
    {
        var key = keySelector(element);
        if (!groups.TryGetValue(key, out var list))
        {
            list = new List<TSource>();
            groups[key] = list;
            orderedKeys.Add(key); // préserver l'ordre d'apparition
        }
        list.Add(element);
    }

    foreach (var key in orderedKeys)
    {
        yield return new Grouping<TKey, TSource>(key, groups[key]);
    }
}

// Implémentation simplifiée de IGrouping
public class Grouping<TKey, TElement> : IGrouping<TKey, TElement>
{
    public TKey Key { get; }
    private readonly List<TElement> _elements;

    public Grouping(TKey key, List<TElement> elements)
    {
        Key = key;
        _elements = elements;
    }

    public IEnumerator<TElement> GetEnumerator() => _elements.GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

# Comment le chaînage fonctionne

Quand on écrit :

```csharp
var result = nombres
    .Where(n => n > 5)
    .Select(n => n * 2)
    .Take(3)
    .ToList();
```

On construit un **pipeline d'itérateurs imbriqués**. Voici ce qui se passe concrètement :

```
ToList() demande des éléments à →
  TakeIterator qui demande des éléments à →
    SelectIterator qui demande des éléments à →
      WhereIterator qui demande des éléments à →
        nombres (la source)
```

Chaque itérateur "tire" les éléments depuis l'itérateur précédent via `MoveNext()`. C'est un modèle **pull-based** (tiré par le consommateur).

```
nombres: [1, 8, 3, 12, 7, 15, 2, 9]

Étape par étape (ce que fait MoveNext à chaque appel) :
─────────────────────────────────────────────────────
1. ToList appelle Take.MoveNext()
2. Take appelle Select.MoveNext()
3. Select appelle Where.MoveNext()
4. Where lit 1 → 1 > 5 ? Non, continue
5. Where lit 8 → 8 > 5 ? Oui → yield return 8
6. Select reçoit 8 → yield return 8 * 2 = 16
7. Take reçoit 16 → taken=1 < 3 → yield return 16
8. ToList ajoute 16

9.  Take appelle Select.MoveNext()
10. Select appelle Where.MoveNext()
11. Where lit 3 → 3 > 5 ? Non, continue
12. Where lit 12 → 12 > 5 ? Oui → yield return 12
13. Select reçoit 12 → yield return 24
14. Take reçoit 24 → taken=2 < 3 → yield return 24
15. ToList ajoute 24

16. Take appelle Select.MoveNext()
17. Select appelle Where.MoveNext()
18. Where lit 7 → 7 > 5 ? Oui → yield return 7
19. Select reçoit 7 → yield return 14
20. Take reçoit 14 → taken=3 → yield return 14, puis yield break
21. ToList ajoute 14

Résultat: [16, 24, 14]
Les éléments 15, 2, 9 n'ont JAMAIS été lus de la source.
```

C'est la beauté de l'exécution différée : on ne parcourt **que ce qui est nécessaire**.

# Catégorisation des opérateurs

| Catégorie | Comportement | Exemples |
|---|---|---|
| **Streaming différé** | Produit les éléments un par un sans tout charger | `Where`, `Select`, `SelectMany`, `Skip`, `Take` |
| **Bufferisé différé** | Doit charger toute la source avant de produire | `OrderBy`, `GroupBy`, `Reverse`, `Join` |
| **Semi-bufferisé** | Maintient un état partiel en mémoire | `Distinct`, `Union`, `Intersect`, `Except` |
| **Immédiat** | Consomme la séquence et retourne un résultat | `ToList`, `Count`, `First`, `Any`, `Sum` |

Comprendre cette catégorisation est essentiel pour estimer la **consommation mémoire** d'un pipeline LINQ.

# Ce qui change dans le vrai .NET

Notre implémentation est fonctionnelle mais simplifiée. L'implémentation officielle dans `System.Linq` ajoute :

1. **Optimisations pour `IList<T>`** : si la source est une liste, `ElementAt`, `Count`, `Last` accèdent par index au lieu d'itérer.

2. **Fusion d'itérateurs** : enchaîner `.Where().Where()` sur le même source crée un seul itérateur combiné au lieu de deux imbriqués.

3. **`Span<T>` et vectorisation** : depuis .NET 8+, certains opérateurs comme `Sum` ou `Min` utilisent SIMD quand la source est un tableau.

4. **`TryGetNonEnumeratedCount`** (.NET 6+) : permet de connaître la taille sans itérer quand la source est une collection.

5. **Arbres d'expressions pour `IQueryable<T>`** : les opérateurs de `Queryable` ne reçoivent pas des `Func<>` mais des `Expression<Func<>>`, ce qui permet de les **analyser** et de les **traduire** (en SQL par exemple) au lieu de les exécuter.

# Résumé

| Concept | Ce qu'on a appris |
|---|---|
| **`yield return`** | Crée un itérateur avec exécution différée, le compilateur génère une machine à états |
| **Méthodes d'extension** | Chaque opérateur LINQ est une méthode d'extension sur `IEnumerable<T>` |
| **Validation séparée** | La méthode publique valide les arguments, une méthode privée contient le `yield` |
| **Pipeline pull-based** | Chaque itérateur "tire" les éléments de l'itérateur précédent via `MoveNext()` |
| **Streaming vs bufferisé** | `Where`/`Select` sont streaming, `OrderBy`/`GroupBy` doivent tout charger |
| **Court-circuit** | `Take`, `First`, `Any` arrêtent l'itération dès que possible |
| **Optimisations** | L'implémentation officielle détecte `ICollection<T>`, fusionne les itérateurs, utilise SIMD |