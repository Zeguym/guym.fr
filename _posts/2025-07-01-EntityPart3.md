---
title: Le Change Tracker (EFcore)
tags: dotnet efcore DbContext change-tracker
series: "Entity Framework"
---

Le **Change Tracker** d’Entity Framework est le composant chargé de surveiller l’état des entités chargées depuis la base de données ou ajoutées au contexte. Il détecte automatiquement les modifications apportées aux propriétés des objets suivis.

<!--more-->
 {% include series.html %}

# Principe

- Lorsqu’une entité est récupérée via le contexte (`DbContext`), le Change Tracker garde en mémoire sa valeur initiale.
- Si une propriété est modifiée, le Change Tracker marque l’entité comme “modifiée”.
- Lors de l’appel à `SaveChanges()`, Entity Framework génère les requêtes SQL nécessaires (INSERT, UPDATE, DELETE) en fonction des changements détectés.

**Avantages :**

- Permet la gestion automatique des mises à jour, suppressions et ajouts.
- Facilite la synchronisation entre le modèle objet et la base de données.

**Exemple :**

```csharp
var client = db.Clients.First();
client.Name = "Nouveau nom"; // Le Change Tracker détecte la modification
db.SaveChanges(); // Génère un UPDATE en base
```

# le AsNoTracking

La méthode `.AsNoTracking()` dans Entity Framework permet d’exécuter une requête sans activer le suivi des entités par le Change Tracker.

- Par défaut, Entity Framework suit toutes les entités récupérées pour détecter les modifications.
- Avec `.AsNoTracking()`, les entités sont chargées en lecture seule : elles ne sont pas surveillées pour les changements.
- Cela améliore les performances, surtout pour les scénarios de lecture où aucune modification ou sauvegarde n’est prévue.

**Exemple :**

```csharp
var produits = db.Products.AsNoTracking().ToList();
```

Ici, les produits sont récupérés sans suivi : toute modification sur ces objets ne sera pas prise en compte lors d’un `SaveChanges()`. Utilisez `.AsNoTracking()` pour les requêtes en lecture seule afin d’optimiser la rapidité et réduire la consommation mémoire.

# ExecuteUpdate, ExecuteDelete et leur pendant asynchrone

Les méthodes `ExecuteUpdate` `ExecuteUpdateAsync`, `ExecuteDelete`et `ExecuteDeleteAsync` d’Entity Framework Core permettent d’exécuter des mises à jour ou suppressions directement en SQL, sans charger les entités en mémoire.

**Conséquences sur le Change Tracker :**

- Ces opérations sont effectuées directement sur la base de données, sans passer par le suivi des entités du contexte.
- Le Change Tracker n’est pas informé des modifications : les entités déjà suivies en mémoire peuvent donc être en désynchronisation avec la base après l’appel à `ExecuteUpdate` ou `ExecuteDelete`.
- Il est recommandé de recharger ou détacher les entités concernées pour éviter des incohérences lors de futures opérations avec le contexte.

## Désynchronisation du contexte

- Si des entités sont déjà chargées et suivies par le contexte (`DbContext`), leurs états en mémoire ne sont pas mis à jour après l’appel à `ExecuteUpdate` ou `ExecuteDelete`.
- Cela peut entraîner des incohérences :
  - Une entité supprimée en base peut rester présente dans le contexte.
  - Une entité modifiée en base peut conserver ses anciennes valeurs en mémoire.

## Risques lors de l’utilisation du contexte partagé

- Dans un contexte _scoped_ (portée par requête http), plusieurs services ou opérations peuvent manipuler les mêmes entités.
- Après un `ExecuteUpdate` ou `ExecuteDelete`, il est possible que des entités suivies soient en conflit avec l’état réel de la base.
- Si un `SaveChanges()` est appelé ensuite, il peut écraser les modifications faites directement en base ou provoquer des erreurs.

## Bonnes pratiques

- Après avoir utilisé `ExecuteUpdate` ou `ExecuteDelete`, il est recommandé :
  - De détacher (`Detach`) les entités concernées du contexte.
  - De recharger les entités depuis la base si elles doivent être utilisées à nouveau.
  - D’éviter d’utiliser ces méthodes si le contexte suit activement des entités du même type dans la même portée.

## Exemple

```csharp
// Suppression directe en base
db.Products.Where(p => p.IsObsolete).ExecuteDelete();

// Les produits obsolètes sont toujours présents dans le Change Tracker
var trackedProducts = db.ChangeTracker.Entries<Product>().ToList();

// Pour éviter les incohérences :
foreach (var entry in trackedProducts)
{
    if (entry.Entity.IsObsolete)
        entry.State = EntityState.Detached;
}
```

`ExecuteUpdate` et `ExecuteDelete` sont puissants pour les opérations bulk, mais ils contournent le Change Tracker. Il faut donc être vigilant sur la cohérence entre le contexte et la base, surtout dans les applications où le contexte est partagé ou injecté en mode scoped.

# Injection de dépendances

Le **Change Tracking** fonctionne grâce au contexte Entity Framework (`DbContext`), qui est généralement injecté dans les classes de votre application (ex : contrôleurs, services) via l’injection de dépendances.

- Lorsqu’un objet `DbContext` est injecté (par exemple via le constructeur), il commence à suivre toutes les entités récupérées ou ajoutées.
- Toutes les modifications apportées aux entités attachées à ce contexte sont détectées automatiquement par le Change Tracker.
- Quand vous appelez `SaveChanges()` sur ce contexte, Entity Framework génère et exécute les requêtes SQL nécessaires pour synchroniser la base avec l’état actuel des objets.

```csharp
public class ClientService
{
    private readonly MyDbContext _context;

    public ClientService(MyDbContext context) // contexte injecté
    {
        _context = context;
    }

    public void UpdateClientName(int id, string newName)
    {
        var client = _context.Clients.Find(id); // entité suivie
        client.Name = newName;                  // modification détectée
        _context.SaveChanges();                 // UPDATE généré
    }
}
```

L’injection du contexte garantit que le Change Tracker suit les entités sur toute la durée de vie du contexte : toutes les modifications, ajouts ou suppressions sont automatiquement prises en compte lors de la sauvegarde.

## Conséquences avec une injection de type Transient

Si le contexte est injecté en Transient, chaque service qui en dépend reçoit sa propre instance. Par exemple, si une API reçoit deux services qui accèdent à la base de données, chacun aura un contexte différent. Cela signifie que les objets suivis par chaque contexte peuvent avoir des états différents.

Si une requête modifie un objet dans un service, l’autre service ne verra pas ce changement avant que les données soient sauvegardées et relues depuis la base.

Injecter le contexte en Transient peut causer des incohérences d’état entre services dans une même requête. Il est généralement recommandé d’utiliser le cycle de vie Scoped pour le contexte afin d’assurer la cohérence des données pendant une requête HTTP.

## Conséquences avec une injection de type Scoped

### Save global

Quand le contexte Entity Framework (DbContext) est de type "scoped" (c’est-à-dire partagé entre plusieurs services pendant une même opération), toutes les modifications suivies par le change tracker sont globales à ce contexte.

**Exemple :**

- Le service A charge des données et commence à les modifier.
- Ensuite, le service A appelle le service B.
- Le service B utilise le même DbContext et appelle `SaveChanges()`.

**Ce qui se passe :**

- Toutes les modifications en cours dans le DbContext (y compris celles faites par le service A) seront sauvegardées dans la base de données, même si le service A n’a pas explicitement demandé la sauvegarde.
- Cela peut entraîner des sauvegardes involontaires de données.

Avec un contexte "scoped", un appel à `SaveChanges()` dans n’importe quel service sauvegarde toutes les modifications suivies par le contexte, pas seulement celles du service qui appelle. Pour éviter cela, il est parfois préférable d’utiliser un contexte "transient" (nouveau contexte pour chaque opération) ou de bien gérer les transactions.

### Désynchronisation

Ici les `ExecuteUpdate` et `ExecuteDelete` peuvent vite être dangereux si le dbContexte est utilisé par plusieurs services. Le DbContext garde en mémoire l’état des entités. Si une mise à jour ou suppression est faite directement en base, le cache peut devenir obsolète, entraînant des erreurs ou des données incorrectes.

### Save tres long\*\*

Lorsque de nombreux objets sont suivis (trackés) par le Change Tracker dans Entity Framework, l’appel à SaveChanges() peut devenir très long.
Voici pourquoi :

Analyse de chaque entité : Avant de générer les requêtes SQL, Entity Framework doit parcourir toutes les entités suivies pour détecter les modifications, ajouts ou suppressions.

- Comparaison des états et des valeurs : Pour chaque entité, le Change Tracker compare les valeurs actuelles avec celles chargées initialement afin de déterminer les colonnes à mettre à jour.
- Génération des requêtes : Plus il y a d’entités modifiées, plus il y a de requêtes à générer et à exécuter.
- Gestion des relations : Si les entités sont liées, Entity Framework doit aussi gérer les mises à jour des relations, ce qui ajoute de la complexité.

Plus le nombre d’entités suivies est élevé, plus le travail du Change Tracker et la génération des requêtes SQL lors du SaveChanges() sont importants, ce qui peut ralentir considérablement l’opération. Il est donc recommandé de ne charger et modifier que les entités nécessaires.

**Il est donc important de mettre le AsNoTracking partout pour les données chargées ne seront pas modifiées.**

# Manipulation manuelle du Change Tracker

Entity Framework permet de contrôler explicitement le suivi des entités via le contexte (`DbContext`). Cela peut être utile pour :

- Attacher une entité non suivie (par exemple, reçue d’une API ou reconstruite).
- Détacher une entité pour arrêter le suivi et éviter des modifications non désirées.

## Attacher une entité

Pour commencer à suivre une entité, utilisez :

```csharp
db.Attach(entity); // Ajoute l'entité au Change Tracker, état = Unchanged
```

Ou pour un état spécifique :

```csharp
db.Entry(entity).State = EntityState.Modified; // Marque l'entité comme modifiée
```

## Détacher une entité

Pour arrêter le suivi d’une entité :

```csharp
db.Entry(entity).State = EntityState.Detached; // L'entité n'est plus suivie
```

## Utilisation typique

- **Attach** : utile pour mettre à jour une entité reçue hors contexte (ex : via un DTO).
- **Detach** : utile après une opération bulk (`ExecuteUpdate`, `ExecuteDelete`) ou pour libérer de la mémoire.

La manipulation manuelle du Change Tracker via `Attach` et `Detach` permet de contrôler précisément quelles entités sont suivies, leur état, et d’éviter des incohérences ou des modifications involontaires lors de l’utilisation avancée d’Entity Framework.

## Suivre les états

Pour interroger les états des entités suivies par le Change Tracker dans Entity Framework, utilise la propriété `ChangeTracker.Entries<T>()` du contexte (`DbContext`). Cela permet d’obtenir la liste des entités suivies et leur état (`Added`, `Modified`, `Deleted`, `Unchanged`, `Detached`).

```csharp
foreach (var entry in db.ChangeTracker.Entries())
{
    Console.WriteLine($"Type: {entry.Entity.GetType().Name}, State: {entry.State}");
}
```

Ou pour un type spécifique :

```csharp
foreach (var entry in db.ChangeTracker.Entries<Product>())
{
    Console.WriteLine($"Produit: {entry.Entity.Name}, État: {entry.State}");
}
```

`ChangeTracker.Entries()` permet d’inspecter à tout moment l’état des entités suivies par le contexte.
