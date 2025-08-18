---
title: "Égalité et Identité en .NET : Comprendre les différences pour les types valeur et référence"
tags: dotnet basics equality identity
---

Comprendre la différence entre **égalité** et **identité** est essentiel dès que l’on commence à utiliser des collections comme `Dictionary`, `HashSet` ou même `List` en .NET. Ce document propose un parcours progressif : d’abord les notions de base (égalité logique vs identité de référence), puis le fonctionnement interne des collections (hash + `Equals`), avant de montrer comment implémenter correctement l’égalité pour les classes et les structs, utiliser des comparateurs externes et éviter les pièges les plus courants.

<!--more-->

# Deux grandes notions : égalité et identité

En .NET, on distingue :

- **Identité (référence)** : deux variables pointent‑elles vers **le même objet** ?
  - Testée par `ReferenceEquals(a, b)` ou, par défaut, `==` sur une `class` non surchargée.
- **Égalité logique (valeur)** : deux objets représentent‑ils **la même chose** selon le métier ?
  - Testée par `Equals`, `IEquatable<T>`, comparateurs, etc.

Les collections basées sur des clés (surtout `Dictionary` et `HashSet`) utilisent **l’égalité logique**, pas (ou pas seulement) l’identité.

# Mécanisme général dans Dictionary / HashSet

Pour rechercher/insérer un élément, ces collections utilisent :

1. **`GetHashCode()`** pour trouver un “bucket”.
2. **`Equals(...)`** (ou `IEqualityComparer<T>.Equals`) pour distinguer les éléments dans ce bucket.

Donc, pour un type `T` utilisé comme **clé** de `Dictionary<TKey, ...>` ou **élément** de `HashSet<T>` :

- `Equals` et `GetHashCode` doivent être **cohérents**.
- Un mauvais `GetHashCode` peut poser des problèmes de **performance**.
- Un mauvais `Equals` peut poser des problèmes **fonctionnels** (clés introuvables, doublons inattendus).

## Contrat fondamental égalité / hash

Pour tout type que l’on souhaite utiliser comme clé ou élément :

1. **Contrat 1 – Cohérence**

- Si `a.Equals(b)` est `true` ⇒ `a.GetHashCode() == b.GetHashCode()` **doit** être vrai.
- L’inverse n’est pas obligatoire : deux objets différents peuvent partager un même hash (collision).

2. **Contrat 2 – Stabilité**

- La valeur de `GetHashCode()` **ne doit pas changer** tant que l’objet est dans un `Dictionary`/`HashSet`.
- Ne pas baser le calcul du hash sur des champs susceptibles d’être modifiés après l’insertion dans la collection.

3. **Contrat 3 – Propriétés d’égalité**

- `Equals` doit être :
  - **Réflexive** : `a.Equals(a)` est `true`.
  - **Symétrique** : si `a.Equals(b)` alors `b.Equals(a)`.
  - **Transitive** : si `a.Equals(b)` et `b.Equals(c)` alors `a.Equals(c)`.

## Classes (types référence) : égalité par défaut vs logique métier

Par défaut, une `class` sans override :

```csharp
class Person
{
    public string Name { get; set; }
}
```

- `Equals` et `==` comparent **les références** :
  - Deux instances avec le même contenu (`Name`) mais créées avec `new` seront **différentes** pour un `Dictionary` ou `HashSet`.

Pour avoir une égalité métier (par valeur), il faut :

1. Implémenter `IEquatable<Person>`.
2. Surcharger `Equals(object)` pour déléguer vers `Equals(Person)`.
3. Surcharger `GetHashCode()` de manière cohérente.

Exemple (égalité basée sur `Email`) :

```csharp
public sealed class Person : IEquatable<Person>
{
    public string Email { get; }
    public string Name  { get; }

    public Person(string email, string name)
    {
        Email = email;
        Name  = name;
    }

    public bool Equals(Person? other) => other is not null && Email == other.Email;

    public override bool Equals(object? obj) => ReferenceEquals(this, obj) || obj is Person other && Equals(other);

    public override int GetHashCode() => Email.GetHashCode(); // ou HashCode.Combine(Email)

    public static bool operator ==(Person? left, Person? right) => Equals(left, right);

    public static bool operator !=(Person? left, Person? right)=> !Equals(left, right);
}
```

Maintenant :

```csharp
var set = new HashSet<Person>();
set.Add(new Person("a@x.com", "Alice"));
set.Add(new Person("a@x.com", "Alice 2"));

set.Count == 1; // égalité logique sur Email
```

## Structs (types valeur) : égalité naturelle et value objects

Pour les `struct` :

- La BCL fournit une égalité par défaut **champ par champ** (mais souvent via réflexion → potentiellement plus lente).
- Pour un **value object** (type valeur métier), il est fortement recommandé :
  - de le rendre **immuable** (`readonly struct`, propriétés en lecture seule),
  - d’implémenter `IEquatable<T>` et `GetHashCode` explicitement.

Exemple :

```csharp
public readonly struct Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        Amount   = amount;
        Currency = currency;
    }

    public bool Equals(Money other) => Amount == other.Amount && Currency == other.Currency;

    public override bool Equals(object? obj) => obj is Money other && Equals(other);

    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
}
```

- Ce `Money` est parfait comme clé de `Dictionary<Money,...>` ou élément de `HashSet<Money>`.

# Comparateur externe : `IEqualityComparer<T>`

Il arrive que l’on **ne puisse pas** ou **ne souhaite pas** modifier le type, par exemple dans les cas suivants :

- Type provenant d’une bibliothèque externe.
- Besoin de plusieurs notions d’égalité (par adresse e‑mail, par identifiant, insensible à la casse, etc.).

Dans ces situations, vous pouvez fournir un comparateur externe :

```csharp
public sealed class PersonEmailComparer : IEqualityComparer<Person>
{
    public bool Equals(Person? x, Person? y) => string.Equals(x?.Email, y?.Email, StringComparison.OrdinalIgnoreCase);

    public int GetHashCode(Person obj) => StringComparer.OrdinalIgnoreCase.GetHashCode(obj.Email);
}

// Usage
var set = new HashSet<Person>(new PersonEmailComparer());
var dict = new Dictionary<Person, Order>(new PersonEmailComparer());
```

- Ici, l’égalité dans la collection est basée **uniquement** sur `Email`, en insensible à la casse.

# Cas particuliers : `Dictionary`, `HashSet`, `List`

- `Dictionary<TKey, TValue>` :

  - Utilise `IEqualityComparer<TKey>` (par défaut : `EqualityComparer<TKey>.Default`, qui s’appuie sur `IEquatable<T>`/`Equals`/`GetHashCode`).
  - Une clé “égale” écrase l’ancienne valeur (`dict[key] = newValue`).

- `HashSet<T>` :

  - Ensemble d’éléments **uniques**.
  - Un `Add` d’un élément égal à un autre existant ne change pas le set (renvoie `false`).

- `List<T>` :
  - Ne se base pas sur le hash pour stocker, mais l’égalité intervient pour : `Contains`, `IndexOf`, `Remove`, etc.
  - Utilise aussi `EqualityComparer<T>.Default`.

Donc le même contrat d’égalité / hash est important **partout**, même si `List<T>` ne se base pas sur `GetHashCode` pour la structure interne.

# Pièges fréquents

1. **Clé mutable** dans un `Dictionary` / `HashSet`

Modification d'un champ utilisé dans `Equals`/`GetHashCode` après insertion. L’objet est “perdu” dans la collection (introuvable, pas supprimable correctement).

2. **Override partiel** (`Equals` sans `GetHashCode`, ou inverse)

Viol du contrat, bugs difficile à diagnostiquer.

3. **Comparer par référence sans le vouloir**

Classe sans override → `Equals` = référence. Deux objets “égaux” métier mais créés avec `new` sont considérés “différents” pour les collections.

4. **Hash code constant ou trop pauvre**

```csharp
public override int GetHashCode() => 1;
```

Fonctionne logiquement, mais :

- toutes les clés/éléments dans le même bucket,
- performances très mauvaises (souvent proche d’une liste chaînée parcourue à chaque accès).

# Résumé pratique / Checklist

1. **Disposez-vous d’une notion claire d’égalité métier ?**

- Qu’est-ce qui fait que deux instances représentent « la même chose » ?
- Même `Email` ? Même `Id` ? Même couple `(X, Y)` ?

2. **Pour une `class` :**

- Implémenter `IEquatable<T>` lorsque le type est maîtrisé.
- Surcharger `Equals(object)` pour déléguer vers `Equals(T)`.
- Surcharger `GetHashCode` de façon cohérente avec `Equals`.
- Facultatif mais recommandé : aligner `==` et `!=` avec `Equals`.

3. **Pour un `struct` :**

- Le rendre immuable (`readonly struct`, propriétés en lecture seule).

4. **Si le type ne peut pas être modifié :**

- Utiliser un `IEqualityComparer<T>` personnalisé pour les `Dictionary` et `HashSet`.

5. **Éviter :**

- De baser `GetHashCode` sur des champs modifiés après insertion.
- De changer la sémantique de l’égalité sans mettre à jour toutes les méthodes concernées.
