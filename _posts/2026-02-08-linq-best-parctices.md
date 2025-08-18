---
title: "LINQ : bonnes pratiques et pièges à éviter"
tags: dotnet basics linq csharp best-practices
series: "Linq"
---

LINQ rend le code expressif et concis, mais il est aussi facile de tomber dans des pièges qui dégradent les performances, provoquent des bugs subtils ou rendent le code difficile à maintenir. Cet article recense les bonnes pratiques à adopter et les erreurs les plus courantes à éviter quand on utilise LINQ au quotidien.

<!--more-->
{% include series.html %}

# Performances

## 1. Préférer `Any()` à `Count() > 0`

`Count()` doit parcourir **toute** la séquence pour compter les éléments. `Any()` s'arrête au **premier élément trouvé**.

```csharp
// ❌ Parcourt toute la collection
if (commandes.Count() > 0) { /* ... */ }

// ✅ S'arrête dès le premier élément
if (commandes.Any()) { /* ... */ }

// Idem avec prédicat
// ❌
if (commandes.Count(c => c.Montant > 100) > 0) { /* ... */ }
// ✅
if (commandes.Any(c => c.Montant > 100)) { /* ... */ }
```

**Exception** : si la source est un `ICollection<T>` (comme `List<T>`), la propriété `.Count` (pas la méthode LINQ `.Count()`) est en O(1). Mais `Any()` reste plus lisible pour exprimer l'intention "y a-t-il au moins un élément ?".

## 2. Ne pas itérer plusieurs fois sur un `IEnumerable<T>`

Chaque appel à `foreach`, `Count()`, `ToList()`, etc. sur un `IEnumerable<T>` **ré-exécute** tout le pipeline. Si la source est une requête Entity Framework, cela signifie **plusieurs aller-retours SQL**.

```csharp
// ❌ Deux itérations = deux exécutions complètes
IEnumerable<Client> actifs = clients.Where(c => c.IsActif);
var nombre = actifs.Count();        // 1ère itération
var liste  = actifs.ToList();       // 2ème itération

// ✅ Matérialiser une seule fois
var actifs = clients.Where(c => c.IsActif).ToList();
var nombre = actifs.Count;          // propriété O(1)
```

ReSharper et Roslyn émettent d'ailleurs un avertissement `CA1851` / "Possible multiple enumerations of IEnumerable" pour ce cas.

## 3. Filtrer le plus tôt possible dans le pipeline

L'ordre des opérateurs a un impact direct sur les performances. Placez les `Where` **avant** les `Select`, `OrderBy` ou `GroupBy` pour réduire le volume de données traitées.

```csharp
// ❌ Trie TOUS les éléments, puis filtre
var resultat = commandes
    .OrderBy(c => c.Date)
    .Where(c => c.Montant > 1000)
    .ToList();

// ✅ Filtre d'abord (moins d'éléments à trier)
var resultat = commandes
    .Where(c => c.Montant > 1000)
    .OrderBy(c => c.Date)
    .ToList();
```

## 4. Chaîner les opérateurs : coût réel de deux `Where` consécutifs

Une question fréquente : est-ce que `.Where(A).Where(B)` est moins performant que `.Where(x => A(x) && B(x))` ?

### Ce qui se passe sous le capot

Chaque appel à `Where` crée un **nouvel itérateur**. Deux `Where` consécutifs produisent donc **deux objets** et **deux niveaux d'imbrication** dans le pipeline :

```csharp
// Deux itérateurs : WhereIterator2 → WhereIterator1 → source
var resultat = personnes
    .Where(p => p.Age >= 18)       // crée WhereIterator1
    .Where(p => p.Nom.Length > 3)  // crée WhereIterator2
    .ToList();

// Un seul itérateur : WhereIterator → source
var resultat = personnes
    .Where(p => p.Age >= 18 && p.Nom.Length > 3) // crée 1 seul WhereIterator
    .ToList();
```

Avec deux `Where`, chaque élément de la source passe par :
1. `WhereIterator1.MoveNext()` → appelle `source.MoveNext()` + évalue le 1er prédicat
2. `WhereIterator2.MoveNext()` → appelle `WhereIterator1.MoveNext()` + évalue le 2ème prédicat

Avec un seul `Where`, chaque élément passe par :
1. `WhereIterator.MoveNext()` → appelle `source.MoveNext()` + évalue les 2 prédicats

### Le coût concret

| Aspect | `.Where(A).Where(B)` | `.Where(A && B)` |
|---|---|---|
| Allocations | 2 itérateurs + 2 delegates | 1 itérateur + 1 delegate |
| Appels `MoveNext()` par élément | 2 | 1 |
| Appels de delegates par élément | 1 ou 2 (court-circuit si A = false) | 1 (court-circuit interne via `&&`) |
| Overhead mémoire | ~2× plus d'objets sur le tas | Minimal |

### L'optimisation de .NET : la fusion d'itérateurs

Bonne nouvelle : l'implémentation officielle de LINQ dans .NET **optimise** ce cas. Quand on enchaîne `.Where().Where()` sur la même source, le runtime fusionne les deux prédicats en un seul itérateur (`CombinedPredicates`) :

```csharp
// .NET détecte l'enchaînement et produit en interne quelque chose comme :
// WhereIterator(source, x => predicate1(x) && predicate2(x))
```

Cette optimisation existe depuis .NET Core, mais **uniquement pour `Where().Where()`**. Elle ne s'applique pas aux combinaisons mixtes comme `Where().Select().Where()`.

### En pratique : quand s'en préoccuper ?

```csharp
// ✅ Parfaitement acceptable pour la lisibilité courante
var commandesValides = commandes
    .Where(c => c.Statut != Statut.Annulee)
    .Where(c => c.Montant > 0)
    .Where(c => c.Date >= dateDebut);
// → fusionné par .NET, coût quasi identique à un seul Where

// ✅ Un seul Where si le prédicat reste lisible
var commandesValides = commandes
    .Where(c => c.Statut != Statut.Annulee && c.Montant > 0 && c.Date >= dateDebut);

// ⚠️ Dans un hot path critique (millions d'itérations par seconde),
//     préférer un seul Where ou une boucle foreach
for (int i = 0; i < data.Length; i++)
{
    if (data[i].A && data[i].B)
        resultats.Add(data[i]);
}
```

### Impact avec `Select` entre deux `Where`

Attention : si un `Select` sépare deux `Where`, la fusion ne s'applique **pas**, et le coût est réel :

```csharp
// ❌ Pas de fusion possible : 3 itérateurs distincts
var resultat = personnes
    .Where(p => p.Age >= 18)          // WhereIterator
    .Select(p => new { p.Nom, p.Age }) // SelectIterator
    .Where(x => x.Nom.Length > 3)     // WhereIterator (sur type anonyme)
    .ToList();

// ✅ Regrouper les filtres avant la projection
var resultat = personnes
    .Where(p => p.Age >= 18 && p.Nom.Length > 3) // 1 seul WhereIterator
    .Select(p => new { p.Nom, p.Age })            // 1 SelectIterator
    .ToList();
```

### Résumé

> Pour du code courant, **la lisibilité prime** : séparer les `Where` en plusieurs lignes est parfaitement acceptable grâce à la fusion automatique. Mais dans les **chemins critiques** ou quand un `Select` s'intercale, regrouper les conditions dans un seul `Where` évite les allocations et les niveaux d'indirection supplémentaires.

## 5. Attention aux allocations cachées

Chaque opérateur LINQ crée un **itérateur** (un objet sur le tas). Dans du code appelé très fréquemment (boucle serrée, hot path), cela peut peser.

```csharp
// ❌ Dans un hot path : allocations à chaque appel
bool Contient(List<int> liste, int valeur)
    => liste.Where(x => x == valeur).Any();

// ✅ Utiliser une boucle classique dans les chemins critiques
bool Contient(List<int> liste, int valeur)
{
    foreach (var x in liste)
        if (x == valeur) return true;
    return false;
}

// ✅ Ou simplement utiliser Contains
bool Contient(List<int> liste, int valeur)
    => liste.Contains(valeur);
```

## 6. Utiliser les bonnes structures de données

LINQ fonctionne sur `IEnumerable<T>`, mais les performances dépendent de la **structure sous-jacente**.

```csharp
// ❌ Recherche dans une List<T> : O(n)
var existe = listeClients.Any(c => c.Id == 42);

// ✅ Utiliser un HashSet ou Dictionary quand on fait beaucoup de recherches
var index = new HashSet<int>(listeClients.Select(c => c.Id));
var existe = index.Contains(42); // O(1)
```

# Pièges avec `IQueryable<T>` (Entity Framework)

## 7. Ne pas faire basculer en mémoire trop tôt

Appeler `ToList()`, `AsEnumerable()` ou `ToArray()` trop tôt dans le pipeline force le chargement **de toutes les données** en mémoire. Les opérateurs suivants s'exécutent alors côté client.

```csharp
// ❌ Charge TOUTE la table, puis filtre en mémoire
var grosClients = dbContext.Clients
    .ToList()                               // SELECT * FROM Clients
    .Where(c => c.ChiffreAffaires > 100000) // filtre en mémoire
    .OrderBy(c => c.Nom)                    // tri en mémoire
    .ToList();

// ✅ Tout est traduit en SQL
var grosClients = dbContext.Clients
    .Where(c => c.ChiffreAffaires > 100000) // WHERE ChiffreAffaires > 100000
    .OrderBy(c => c.Nom)                    // ORDER BY Nom
    .ToList();                              // une seule requête SQL optimisée
```

## 8. Éviter les méthodes C# non traduisibles en SQL

Les providers `IQueryable` (Entity Framework) ne peuvent traduire que certaines expressions. Appeler une méthode C# arbitraire dans un `Where` ou `Select` provoque soit une erreur, soit une évaluation côté client inattendue.

```csharp
// ❌ EF ne peut pas traduire FormatNom() en SQL
var noms = dbContext.Clients
    .Where(c => FormatNom(c.Nom).StartsWith("A"))
    .ToList();

// ✅ Option 1 : utiliser les expressions que EF sait traduire
var noms = dbContext.Clients
    .Where(c => c.Nom.StartsWith("A"))
    .ToList();

// ✅ Option 2 : basculer en mémoire APRÈS le filtre SQL
var noms = dbContext.Clients
    .Where(c => c.Nom.StartsWith("A"))  // filtre SQL
    .AsEnumerable()                      // bascule en mémoire
    .Where(c => FormatNom(c.Nom).Length > 5) // filtre C#
    .ToList();
```

## 9. Projeter uniquement les colonnes nécessaires

Sans `Select`, Entity Framework charge **toutes les colonnes** de l'entité.

```csharp
// ❌ Charge toutes les colonnes de Client (y compris les BLOBs, etc.)
var noms = dbContext.Clients
    .Where(c => c.IsActif)
    .ToList()
    .Select(c => c.Nom);

// ✅ Ne charge que la colonne Nom (SELECT Nom FROM ...)
var noms = dbContext.Clients
    .Where(c => c.IsActif)
    .Select(c => c.Nom)
    .ToList();
```

# Pièges avec l'exécution différée

## 10. La source peut changer entre la définition et l'exécution

L'exécution différée signifie que la requête n'est évaluée que quand on itère. Si la source change entre-temps, les résultats reflètent l'état **au moment de l'itération**, pas de la définition.

```csharp
var liste = new List<int> { 1, 2, 3 };
var grands = liste.Where(n => n > 1);

liste.Add(10);     // modification APRÈS la définition de la requête
liste.Remove(2);   // suppression d'un élément

var resultat = grands.ToList(); // [3, 10] — reflète l'état actuel
```

Cela peut être **voulu** (la requête est toujours "fraîche") ou être un **bug** (on s'attendait à un snapshot).

## 11. Les closures capturent la variable, pas la valeur

```csharp
// ❌ Piège classique : toutes les actions utilisent la valeur finale de i
var actions = new List<Func<int>>();
for (int i = 0; i < 5; i++)
{
    actions.Add(() => i * 10);
}
// actions[0]() == 50, actions[1]() == 50, ..., actions[4]() == 50
// Car i vaut 5 après la boucle

// ✅ Capturer une copie locale
for (int i = 0; i < 5; i++)
{
    var copie = i;
    actions.Add(() => copie * 10);
}
// actions[0]() == 0, actions[1]() == 10, ..., actions[4]() == 40
```

> Avec `foreach`, ce piège n'existe plus depuis C# 5 : la variable d'itération est capturée correctement à chaque itération.

## 12. `SingleOrDefault` vs `FirstOrDefault`

Ces deux méthodes ne sont **pas interchangeables** :

| Méthode | 0 élément | 1 élément | 2+ éléments |
|---|---|---|---|
| `First()` | ❌ Exception | ✅ Retourne l'élément | ✅ Retourne le premier |
| `FirstOrDefault()` | ✅ Retourne `default` | ✅ Retourne l'élément | ✅ Retourne le premier |
| `Single()` | ❌ Exception | ✅ Retourne l'élément | ❌ Exception |
| `SingleOrDefault()` | ✅ Retourne `default` | ✅ Retourne l'élément | ❌ Exception |

```csharp
// Quand on s'attend à 0 ou 1 résultat (et que >1 est une erreur)
var client = dbContext.Clients.SingleOrDefault(c => c.Email == email);

// Quand on veut juste le premier, peu importe s'il y en a plusieurs
var derniereCommande = dbContext.Commandes
    .OrderByDescending(c => c.Date)
    .FirstOrDefault();
```

**Côté SQL** : `SingleOrDefault` doit vérifier qu'il n'y a pas de second résultat, ce qui peut être marginalement plus coûteux.

# Lisibilité et maintenabilité

## 13. Éviter les pipelines trop longs

Un pipeline LINQ de 15 opérateurs enchaînés devient illisible. Découper en étapes nommées.

```csharp
// ❌ Pipeline illisible
var resultat = commandes
    .Where(c => c.Date >= debut && c.Date <= fin)
    .Where(c => c.Statut != Statut.Annulee)
    .GroupBy(c => c.ClientId)
    .Select(g => new { ClientId = g.Key, Total = g.Sum(c => c.Montant) })
    .Where(x => x.Total > 10000)
    .OrderByDescending(x => x.Total)
    .Take(10)
    .ToList();

// ✅ Étapes nommées qui documentent l'intention
var commandesPeriode = commandes
    .Where(c => c.Date >= debut && c.Date <= fin)
    .Where(c => c.Statut != Statut.Annulee);

var topClients = commandesPeriode
    .GroupBy(c => c.ClientId)
    .Select(g => new { ClientId = g.Key, Total = g.Sum(c => c.Montant) })
    .Where(x => x.Total > 10000)
    .OrderByDescending(x => x.Total)
    .Take(10)
    .ToList();
```

Comme les variables intermédiaires sont des `IEnumerable`/`IQueryable`, il n'y a **aucun coût** supplémentaire : le pipeline final est identique.

## 14. Nommer les types anonymes quand ils sont réutilisés

```csharp
// ❌ Type anonyme difficile à passer en paramètre ou retourner
var stats = commandes
    .GroupBy(c => c.ClientId)
    .Select(g => new { ClientId = g.Key, Total = g.Sum(c => c.Montant), Nombre = g.Count() });

// ✅ Utiliser un record pour un type nommé, réutilisable, testable
public record ClientStats(int ClientId, decimal Total, int Nombre);

var stats = commandes
    .GroupBy(c => c.ClientId)
    .Select(g => new ClientStats(g.Key, g.Sum(c => c.Montant), g.Count()));
```

## 15. Préférer les opérateurs LINQ aux réinventions manuelles

```csharp
// ❌ Réinventer la roue
var max = int.MinValue;
foreach (var c in commandes)
    if (c.Montant > max) max = c.Montant;

// ✅ Utiliser l'opérateur dédié
var max = commandes.Max(c => c.Montant);

// ❌ Construire un dictionnaire manuellement
var dict = new Dictionary<int, Client>();
foreach (var c in clients)
    dict[c.Id] = c;

// ✅
var dict = clients.ToDictionary(c => c.Id);
```

# Pièges spécifiques

## 16. `ToDictionary` lance une exception si les clés ne sont pas uniques

```csharp
// ❌ Exception si deux clients ont le même email
var parEmail = clients.ToDictionary(c => c.Email);

// ✅ Si les doublons sont possibles, utiliser ToLookup ou GroupBy
var parEmail = clients.ToLookup(c => c.Email); // ILookup<string, Client>

// ✅ Ou gérer explicitement les doublons
var parEmail = clients
    .GroupBy(c => c.Email)
    .ToDictionary(g => g.Key, g => g.First());
```

## 17. `Distinct()` utilise l'égalité par défaut du type

Pour les types référence (classes), `Distinct()` compare par **référence** sauf si `Equals`/`GetHashCode` sont redéfinis.

```csharp
// ❌ Ne supprime PAS les doublons logiques (comparaison par référence)
var villes = clients.Select(c => new Ville(c.CodePostal, c.NomVille)).Distinct();

// ✅ Option 1 : utiliser un record (égalité structurelle automatique)
public record Ville(string CodePostal, string Nom);
var villes = clients.Select(c => new Ville(c.CodePostal, c.NomVille)).Distinct();

// ✅ Option 2 : utiliser DistinctBy (.NET 6+)
var villes = clients.DistinctBy(c => c.CodePostal);

// ✅ Option 3 : fournir un IEqualityComparer<T>
var villes = clients
    .Select(c => new Ville(c.CodePostal, c.NomVille))
    .Distinct(new VilleComparer());
```

## 18. `OrderBy` n'est pas stable avec `IQueryable`

En LINQ to Objects, `OrderBy` est un tri **stable** (les éléments égaux conservent leur ordre relatif). Mais en SQL, l'ordre des lignes avec la même clé de tri est **indéterminé**.

```csharp
// ❌ L'ordre des clients ayant le même nom est imprévisible en SQL
var clients = dbContext.Clients.OrderBy(c => c.Nom).ToList();

// ✅ Toujours ajouter un critère de départage déterministe
var clients = dbContext.Clients
    .OrderBy(c => c.Nom)
    .ThenBy(c => c.Id)  // Id unique = ordre déterministe
    .ToList();
```

## 19. `string.Contains` est sensible à la casse en C# mais pas toujours en SQL

```csharp
// En LINQ to Objects : sensible à la casse
var resultat = noms.Where(n => n.Contains("alice")); // ne trouve pas "Alice"

// En EF Core avec SQL Server : insensible à la casse (collation par défaut)
var resultat = dbContext.Clients
    .Where(c => c.Nom.Contains("alice")); // TROUVE "Alice" (dépend de la collation)

// ✅ Être explicite sur la comparaison
// En mémoire :
var resultat = noms
    .Where(n => n.Contains("alice", StringComparison.OrdinalIgnoreCase));

// En EF Core (.NET 8+) :
var resultat = dbContext.Clients
    .Where(c => EF.Functions.Like(c.Nom, "%alice%"));
```

# Aide-mémoire

| Règle | Pourquoi |
|---|---|
| `Any()` plutôt que `Count() > 0` | Court-circuit, performance |
| Matérialiser une seule fois | Éviter les itérations multiples et les requêtes SQL dupliquées |
| Filtrer avant de trier/projeter | Réduire le volume de données à traiter |
| `ToList()` le plus tard possible (EF) | Garder l'exécution côté serveur |
| `Select` pour projeter les colonnes (EF) | Ne charger que ce qui est nécessaire |
| Nommer les étapes longues | Lisibilité et maintenabilité |
| `SingleOrDefault` vs `FirstOrDefault` | Sémantique différente, choisir avec intention |
| `ThenBy` pour un tri déterministe | Éviter les ordres imprévisibles en SQL |
| `DistinctBy` ou records pour `Distinct` | L'égalité par défaut des classes est par référence |
| Boucle classique dans les hot paths | Éviter les allocations des itérateurs LINQ |