---
title: Le Database Context  (EFcore)
tags: dotnet efcore DbContext 
series: "Entity Framework"
---

Dans cette deuxième partie consacrée à Entity Framework, l’accent est mis sur le rôle central du `DbContext` et sur la manière de gérer correctement son cycle de vie. Sont notamment abordés les raisons pour lesquelles il n’est pas thread-safe, les moyens d’éviter les problèmes de concurrence, les stratégies d’injection adaptées (scoped, transient, singleton) en fonction du type d’application, ainsi que les cas d’usage d’un `DbContextFactory` pour créer des contextes à la demande. Enfin, l’impact du `DbContext` sur les connexions à la base de données est présenté, afin de prévenir la saturation du serveur et de garantir des applications plus stables et performantes.

<!--more-->
 {% include series.html %}


# Concurrence

Le `DbContext` d’Entity Framework n’est **pas thread-safe** : il ne doit pas être partagé entre plusieurs threads simultanément.

**Pourquoi ?**

- Le `DbContext` maintient un état interne (Change Tracker, transactions, connexions) qui n’est pas conçu pour être modifié par plusieurs threads en même temps.
- Un accès concurrent peut provoquer des exceptions, des corruptions de données ou des incohérences dans le suivi des entités.

**Bonne pratique :**

- Utiliser une instance de `DbContext` par thread ou par requête (mode scoped en ASP.NET Core).
- Ne jamais partager le même contexte entre plusieurs threads : chaque tâche parallèle doit disposer de sa propre instance.

Le `DbContext` doit être utilisé dans un contexte mono-threadé. Il ne peut exécuter qu'une seule requête à la fois (de l'envoi du SQL à la réception complète des résultats).

# Cloner un contexte

En Entity Framework, il n’est **pas possible de cloner directement un `DbContext`**. Le contexte contient des états internes (Change Tracker, connexions, transactions) qui ne peuvent pas être dupliqués proprement.

**Pour obtenir un contexte équivalent, il convient de :**

- Créer une nouvelle instance du même type de contexte, avec la même configuration (par exemple : chaîne de connexion, options).
- Les entités suivies par le Change Tracker du contexte original ne seront pas suivies dans le nouveau contexte.

**Exemple :**

```csharp
var newContext = new MyDbContext(existingContext.Options);
```

Toutefois, les entités attachées au premier contexte ne sont pas automatiquement attachées au nouveau. Pour transférer des entités, il est nécessaire de les attacher manuellement au nouveau contexte :

```csharp
newContext.Attach(entity);
```

# Injection

## Injection directe du DbContext

Voici comment injecter le `DbContext` dans une application .NET, selon les différents cycles de vie :

### 1. **Scoped** (recommandé pour les applications web)

- **Définition** : une instance du contexte est créée et partagée pour chaque requête HTTP.
- **Configuration** :
  ```csharp
  services.AddDbContext<MyDbContext>(options =>
      options.UseSqlServer(connectionString));
  ```
- **Avantages** :
  - garantit la cohérence des données et du Change Tracker pendant toute la requête ;
  - limite les conflits entre utilisateurs ou requêtes concurrentes ;
  - constitue l’option recommandée pour ASP.NET Core.
- **Injection** :
  ```csharp
  public class MyService
  {
      private readonly MyDbContext _context;
      public MyService(MyDbContext context) { _context = context; }
  }
  ```

### 2. **Transient**

- **Définition** : une nouvelle instance du contexte est créée à chaque injection.
- **Configuration** :
  ```csharp
  services.AddDbContext<MyDbContext>(options =>
      options.UseSqlServer(connectionString), ServiceLifetime.Transient);
  ```
- **Avantages** :
  - utile pour des opérations très courtes ou isolées ;
  - chaque service reçoit son propre contexte.
- **Inconvénients** :
  - risque d’incohérence si plusieurs services manipulent les mêmes données dans une même requête ;
  - efficacité moindre pour la gestion des transactions partagées.

### 3. **Singleton** (à éviter)

- **Définition** : une seule instance du contexte est partagée pour toute la durée de vie de l’application.
- **Configuration** :
  ```csharp
  services.AddDbContext<MyDbContext>(options =>
      options.UseSqlServer(connectionString), ServiceLifetime.Singleton);
  ```
- **Inconvénients** :
  - Le contexte n’est pas thread-safe : risque de corruption de données et d’exceptions.
  - Ne convient pas à la plupart des scénarios.

### Choix du cycle de vie

- **Scoped** est le choix recommandé pour les applications web (ASP.NET Core), car il garantit la cohérence et la sécurité du suivi des entités pendant chaque requête.
- **Transient** peut être utilisé pour des traitements ponctuels, sous réserve d’une attention particulière portée aux risques d’incohérence.
- **Singleton** doit être évité avec `DbContext`.

De manière générale, il est recommandé d’utiliser le cycle de vie _scoped_ pour injecter le `DbContext` dans les applications web. Cette approche favorise la cohérence, la sécurité et les performances du suivi des entités avec Entity Framework.

## DbContextFactory

Le `DbContextFactory` est un mécanisme d’Entity Framework Core permettant de créer des instances de `DbContext` à la demande, plutôt que de les injecter directement.  
Il est particulièrement utile dans les scénarios où :

- un contexte est requis en dehors du cycle de vie classique (par exemple : tâches parallèles, services en arrière-plan ou composants non gérés par l’injection de dépendances) ;
- il est nécessaire de garantir qu’une nouvelle instance du contexte est créée à chaque utilisation, évitant ainsi les problèmes de concurrence ou de partage d’état.

Pour injecter `IDbContextFactory<TContext>` dans une application .NET (par exemple ASP.NET Core), il suffit d’ajouter la ligne suivante dans la configuration des services (généralement dans `Program.cs` ou `Startup.cs`) :

```csharp
services.AddDbContextFactory<MyDbContext>(options =>
  options.UseSqlServer(connectionString));
```

L’appel à `AddDbContextFactory` dans la configuration des services permet d’injecter `IDbContextFactory<TContext>` dans les classes et de créer des contextes à la demande.

**Utilisation typique :**

```csharp
public class MyService
{
    private readonly IDbContextFactory<MyDbContext> _factory;

    public MyService(IDbContextFactory<MyDbContext> factory)
    {
        _factory = factory;
    }

    public void DoWork()
    {
        using var context = _factory.CreateDbContext();
        // Utilise le contexte ici
    }
}
```

**Avantages :**

- chaque appel à `CreateDbContext()` retourne une nouvelle instance, isolée et thread-safe ;
- ce mécanisme est idéal pour les traitements parallèles ou asynchrones.

**Résumé :**  
Le `DbContextFactory` permet de créer des contextes à la volée, assurant une gestion maîtrisée et flexible du cycle de vie du contexte Entity Framework Core.

## DbContext et Connections

Le nombre d’instances de `DbContext` dans une application influence directement le nombre de connexions ouvertes vers la base de données :

- **chaque `DbContext` peut ouvrir sa propre connexion** à la base, généralement lors de la première requête ;
- si un grand nombre d’instances de `DbContext` sont créées (par exemple via une injection en mode transient ou une création manuelle répétée), il est possible d’ouvrir de nombreuses connexions simultanées ;
- les connexions sont gérées par un pool (connection pooling), mais il existe une limite : si trop de contextes sont actifs en même temps, le nombre maximal de connexions autorisées par le serveur SQL peut être atteint ;
- cette situation peut entraîner des erreurs du type « Too many connections », ralentir l’application ou provoquer des blocages.

**Bonne pratique :**

- utiliser le cycle de vie _scoped_ pour limiter le nombre de contextes vivants en même temps ;
- libérer rapidement les contextes (et donc les connexions) après usage ;
- éviter de créer des contextes inutiles ou de les conserver trop longtemps en mémoire.

**Résumé :**  
Plus le nombre d’instances de `DbContext` vivantes est élevé, plus le nombre de connexions ouvertes augmente ; il est donc nécessaire de gérer leur cycle de vie avec attention afin d’éviter la saturation du serveur de base de données.
