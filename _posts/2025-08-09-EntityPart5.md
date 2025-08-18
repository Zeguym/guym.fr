---
title: Manipuler le Change Tracker (EFcore)
tags: dotnet efcore change-tracker 
series: "Entity Framework"
---

Ef core et le change tracker : Quand et pourquoi utiliser Attach, Update ou Entry pour rattacher des entités au DbContext

<!--more-->
 {% include series.html %}
# 1. Rôle du Change Tracker

Le Change Tracker d’Entity Framework Core a pour mission de suivre l’état des entités manipulées par un `DbContext`.  
Lorsque vous récupérez une entité via ce contexte (sans `AsNoTracking()`), EF Core :

- commence à suivre cette entité automatiquement,
- détecte les modifications apportées à ses propriétés,
- génère les commandes SQL appropriées lors de l’appel à `SaveChanges()`.

Dans ce cas, **il n’est pas nécessaire d’attacher explicitement l’entité**.

# 2. Situations où l’attachement explicite est nécessaire

Vous devez utiliser `Attach`, `Update` ou manipuler l’état via `Entry` lorsque vous travaillez avec des entités **détachées**, c’est‑à‑dire des objets que le `DbContext` ne suit pas encore.

## 2.1. Mise à jour d’une entité reconstruite (DTO → entité)

Scénario : vous recevez un DTO depuis un client et vous construisez une entité avec un identifiant déjà existant, sans l’avoir relue depuis la base dans ce `DbContext`.

```csharp
var product = new Product
{
    Id = dto.Id,
    Name = dto.Name,
    Price = dto.Price
};

// Indiquer à EF que cette entité existe déjà en base
_context.Attach(product);
_context.Entry(product).State = EntityState.Modified;

await _context.SaveChangesAsync();
```

Ici, l’attachement est indispensable, car le contexte ne connaît pas encore cette instance.

> Variante : `_context.Update(product)` réalise un `Attach` et marque l’ensemble du graphe comme `Modified`.

## 2.2. Mise à jour partielle de propriétés

Si vous souhaitez mettre à jour uniquement certaines propriétés :

```csharp
var product = new Product { Id = dto.Id };
_context.Attach(product);

product.Price = dto.Price;

_context.Entry(product).Property(p => p.Price).IsModified = true;

await _context.SaveChangesAsync();
```

L’appel à `Attach` permet au Change Tracker de suivre l’objet et de savoir quelles propriétés doivent être persistées.

## 2.3. Reprise d’une entité issue d’une requête `AsNoTracking()`

Les requêtes `AsNoTracking()` retournent des entités non suivies. Pour les modifier ensuite, il faut les rattacher au contexte :

```csharp
var product = await _context.Products
    .AsNoTracking()
    .FirstOrDefaultAsync(p => p.Id == id);

// Modifications
product.Price = newPrice;

// Rattachement au contexte
_context.Attach(product);
_context.Entry(product).State = EntityState.Modified;

await _context.SaveChangesAsync();
```

Sans cette étape, `SaveChanges()` n’a aucun effet sur cette entité.

## 2.4. Gestion des relations avec des entités existantes

Lorsqu’une nouvelle entité doit être reliée à une entité déjà existante, connue uniquement par son identifiant :

```csharp
var order = new Order { /* ... */ };

var customer = new Customer { Id = customerId };
_context.Attach(customer); // entité existante

order.Customer = customer;

_context.Orders.Add(order);
await _context.SaveChangesAsync();
```

L’attachement du `Customer` indique à EF Core qu’il ne doit pas insérer un nouvel enregistrement pour ce client.

## 2.5. Réattachement d’un graphe provenant d’un autre contexte ou d’un autre processus

Si vous désérialisez un graphe d’entités obtenu ailleurs (autre `DbContext`, autre couche, autre service), il est nécessaire de le rattacher :

```csharp
_context.Attach(order);
// éventuellement, pour forcer une mise à jour globale :
_context.Entry(order).State = EntityState.Modified;

await _context.SaveChangesAsync();
```

# 3. Situations où l’attachement n’est pas nécessaire

Vous n’avez **pas** besoin d’appeler `Attach` lorsque :

- l’entité a été chargée par le même `DbContext` **sans** `AsNoTracking()` :

```csharp
var product = await _context.Products.FirstAsync(p => p.Id == id);
product.Price = 10;

await _context.SaveChangesAsync(); // EF détecte la modification automatiquement
```

- vous ajoutez une nouvelle entité :

```csharp
_context.Products.Add(new Product { Name = "X" });
// `Add` attache l’entité avec l’état `Added`
await _context.SaveChangesAsync();
```

- vous ne faites que lire les données, sans intention de les modifier.

# 4. Synthèse des bonnes pratiques

En résumé, vous devez attacher une entité (via `Attach`, `Update` ou `Entry(entity).State = …`) **uniquement lorsque** :

1. L’objet n’a pas été chargé par le `DbContext` courant (ou l’a été via `AsNoTracking()`), **et**
2. Vous souhaitez qu’il participe à `SaveChanges()` (INSERT, UPDATE, DELETE, modification de relations, etc.).

Règle pratique :

- Entité chargée par le contexte (sans `AsNoTracking`) → **pas d’`Attach`**.
- Entité reconstruite à partir d’un DTO, avec un Id existant → **`Attach` ou `Update`**.
- Entité retournée par une requête `AsNoTracking()` que vous voulez modifier → **`Attach` avant la mise à jour**.
- Entité utilisée uniquement comme “référence” attachée à un nouvel objet (relation, clé étrangère) → **`Attach` la référence** pour éviter les `INSERT` involontaires.
