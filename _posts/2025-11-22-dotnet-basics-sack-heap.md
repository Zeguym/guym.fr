---
title: 'Pile et Tas en .NET : Comprendre la gestion mémoire des types valeur et référence'
tags: dotnet basics stack heap
---

Comprendre la différence entre pile (stack) et tas (heap) est essentiel pour bien raisonner sur la mémoire en .NET, surtout quand on manipule types valeur et types référence. Comprendre où sont stockées les données aide à éviter des bugs subtils et à écrire du code plus performant.

<!--more-->

# Définitions 

- **Pile (stack)** : zone mémoire LIFO, très rapide, gérée automatiquement par le runtime à l’entrée/sortie des méthodes.
- **Tas (heap)** : grande zone mémoire pour les objets “dynamiques”, gérée par le **Garbage Collector (GC)**.

En C#, ce qui détermine où va quoi, c’est surtout **type valeur vs type référence**.

## Types valeur (généralement sur la pile)**

- `int`, `bool`, `double`, `decimal`, `DateTime`, `struct`, `enum`, `record struct`…
- Lorsqu’on déclare une variable locale de type valeur :

```csharp
int x = 42;              // valeur sur la pile
DateTime now = DateTime.Now;
Point p = new Point(1, 2); // struct
```

la **valeur elle-même** est en général sur la pile (ou intégrée dans un objet sur le tas si c’est un champ).

Caractéristiques importantes :

- **Copie par valeur** : `var y = x;` fait une copie indépendante.
- Durée de vie liée au contexte : une variable locale “meurt” à la fin de la méthode (ou du scope).

## Types référence (stockés sur le tas)

- `class`, `string`, `object`, tableaux (`int[]`), `List<T>`, `record class`, etc.
- Lorsque vous écrivez :

```csharp
Person p = new Person("Alice");
```

- La **référence** (`p`) est sur la pile (variable locale),
- L’**objet** `Person` est sur le **tas** (heap).

Caractéristiques importantes :

- **Copie de référence** : `var p2 = p;` → deux variables pointent vers le même objet.
- Durée de vie contrôlée par le **GC** : l’objet reste tant qu’il est accessible par au moins une référence.

# Ce qui est vraiment piégeux pour un dev

## Confondre copie de valeur et copie de référence

```csharp
var p1 = new Person("Alice");
var p2 = p1;   // même objet sur le tas
p2.Name = "Bob";
// p1.Name == "Bob"
```

Alors que pour un type valeur :

```csharp
int a = 10;
int b = a;   // copie
b = 20;
// a == 10, b == 20
```

## Structs (types valeur) qui ressemblent à des objets

```csharp
public struct MutablePoint { public int X; public int Y; }

var points = new MutablePoint[1];
points[0] = new MutablePoint { X = 1, Y = 2 };

var p = points[0]; // COPIE de la valeur
p.X = 10;
// points[0].X est toujours 1
```

- Le tableau est un **objet sur le tas**,
- Mais chaque élément est un **type valeur** copié à la lecture.

## Performance : ce qu’il faut retenir (sans micro-optimiser)

- **Pile :**
  - Allocation/libération très rapide.
  - Taille limitée (stack overflow possible avec trop de récursion ou de grosses structures locales).
- **Tas :**
  - Allocation un peu plus coûteuse, libération gérée par le GC.
  - Très flexible, adapté aux objets qui vivent plus longtemps ou changent de taille (listes, etc.).

En pratique :

- Utilise les **structs petits et immuables** pour des “value objects” simples.
- Utilise des **classes** pour tout ce qui est plus gros, complexe, partagé.

## Idées fausses fréquentes

- “Les types valeur sont toujours sur la pile” → non :
  - Un champ `int` dans une classe est stocké **à l’intérieur de l’objet sur le tas**.
- “Un objet disparaît à la fin de la méthode” → non :
  - S’il est retourné ou stocké ailleurs, il continue d’exister sur le tas.



# Pile et appel de méthode

À chaque appel de méthode, le runtime crée un **stack frame** (cadre de pile) :

- Adresse de retour (où revenir après la méthode).
- **Paramètres** de la méthode.
- **Variables locales**.
- Données techniques (registres, etc.).

Quand la méthode se termine :

- Son **frame est dépilé**, toutes ses variables locales disparaissent.
- L’exécution reprend à l’adresse de retour (frame appelant).

Schéma logique (du bas vers le haut) :

- Frame de `Main`
- → Frame de `Foo`
- → Frame de `Bar`
- etc.

- **Pile (stack)** : variables locales, paramètres, types valeur → rapide, durée de vie courte, copie par valeur.
- **Tas (heap)** : objets (classes, tableaux, strings, listes) → partagés par référence, gérés par le GC.
- La notion clé côté C# n’est pas “où c’est physiquement en mémoire”, mais :
  - **Type valeur** → copie indépendante.
  - **Type référence** → plusieurs variables peuvent pointer vers le même objet.

## Paramètres et variables locales : valeur vs référence**

Exemple 1 – type valeur :

```csharp
void Main()
{
    int x = 10;   // x sur la pile (valeur)
    Foo(x);
}

void Foo(int value)
{
    int y = value + 1;  // y sur la pile (valeur)
}
```

- `x`, `value`, `y` : valeurs sur les différents frames de pile.
- `value` reçoit une **copie** de `x`.

Exemple 2 – type référence :

```csharp
class Person { public string Name { get; set; } }

void Main()
{
    var p = new Person { Name = "Alice" }; // objet sur le tas
    Foo(p);
}

void Foo(Person person)
{
    person.Name = "Bob";
}
```

- `p` et `person` sont des **références** sur la pile.
- Les deux références pointent vers **le même objet** `Person` sur le tas.
- Modifier `person.Name` modifie l’objet, visible aussi via `p`.

## Structs et empilement

```csharp
struct Point { public int X; public int Y; }

void Main()
{
    Point p = new Point { X = 1, Y = 2 }; // p (valeur) sur la pile
    Foo(p);
}

void Foo(Point point)
{
    point.X = 10; // modifie la copie locale
}
```

- `Point` est un **type valeur** :
  - `p` est stocké dans le frame de `Main`.
  - `point` est une **copie** dans le frame de `Foo`.
- À la fin de `Foo`, `point` disparaît, `p` reste inchangé.

Si le struct est champ d’un objet :

```csharp
class Holder { public Point P; }

void Main()
{
    var h = new Holder { P = new Point { X = 1, Y = 2 } };
    Foo(h);
}

void Foo(Holder holder)
{
    holder.P.X = 10;
}
```

- `Holder` est sur le **tas**, champs inclus.
- `holder` est une référence sur la pile.
- `holder.P.X = 10` modifie le champ de l’objet sur le tas.

## Appels imbriqués et dépassement de pile

Avec une récursion profonde :

```csharp
void Recurse(int n)
{
    if (n == 0) return;
    Recurse(n - 1);
}
```

- Chaque appel crée un **nouveau frame** sur la pile.
- Si la profondeur est trop grande → `StackOverflowException` (pile physique saturée).
