---
title: Entity Framework Part 2
tags: dotnet efcore DbContext
---

# Le Database Context 
## Concurence
Le `DbContext` d’Entity Framework n’est **pas thread-safe** : il ne doit pas être partagé entre plusieurs threads simultanément.

**Pourquoi ?**
- Le `DbContext` maintient un état interne (Change Tracker, transactions, connexions) qui n’est pas conçu pour être modifié par plusieurs threads en même temps.
- Un accès concurrent peut provoquer des exceptions, des corruptions de données ou des incohérences dans le suivi des entités.

**Bonne pratique :**
- Utilise une instance de `DbContext` par thread ou par requête (mode scoped en ASP.NET Core).
- Ne partage jamais le même contexte entre plusieurs threads : chaque tâche parallèle doit créer sa propre instance.
 
Le `DbContext` doit être utilisé dans un contexte mono-threadé. Il ne peut exécuter qu'une seule requête a la fois ( de l'envoi du sql à la reception complète des résultats)

## "Clonner" un contexte
En Entity Framework, il n’est **pas possible de cloner directement un `DbContext`**. Le contexte contient des états internes (Change Tracker, connexions, transactions) qui ne peuvent pas être dupliqués proprement.

**Pour dupliquer un contexte, il faut :**
- Créer une nouvelle instance du même type de contexte, avec la même configuration (ex : chaîne de connexion, options).
- Les entités suivies par le Change Tracker du contexte original ne seront pas suivies dans le nouveau contexte.

**Exemple :**
```csharp
var newContext = new MyDbContext(existingContext.Options);
```
Mais attention : Les entités attachées au premier contexte ne sont pas automatiquement attachées au nouveau. Pour transférer des entités, il faut les attacher manuellement au nouveau contexte :
```csharp
newContext.Attach(entity);
```
# Injection

## Injection directe du DB context
Voici comment injecter le `DbContext` dans une application .NET, selon les différents cycles de vie :

### 1. **Scoped** (recommandé pour les applications web)

- **Définition** : Une instance du contexte est créée et partagée pour chaque requête HTTP.
- **Configuration** :
    ```csharp
    services.AddDbContext<MyDbContext>(options =>
        options.UseSqlServer(connectionString));
    ```
- **Avantages** :
    - Garantit la cohérence des données et du Change Tracker pendant toute la requête.
    - Évite les conflits entre utilisateurs ou requêtes concurrentes.
    - Recommandé pour ASP.NET Core.
- **Injection** :
    ```csharp
    public class MyService
    {
        private readonly MyDbContext _context;
        public MyService(MyDbContext context) { _context = context; }
    }
    ```

### 2. **Transient**

- **Définition** : Une nouvelle instance du contexte est créée à chaque injection.
- **Configuration** :
    ```csharp
    services.AddDbContext<MyDbContext>(options =>
        options.UseSqlServer(connectionString), ServiceLifetime.Transient);
    ```
- **Avantages** :
    - Utile pour des opérations très courtes ou isolées.
    - Chaque service reçoit son propre contexte.
- **Inconvénients** :
    - Risque d’incohérence si plusieurs services manipulent les mêmes données dans une même requête.
    - Moins efficace pour la gestion des transactions partagées.

### 3. **Singleton** (à éviter)

- **Définition** : Une seule instance du contexte est partagée pour toute la durée de vie de l’application.
- **Configuration** :
    ```csharp
    services.AddDbContext<MyDbContext>(options =>
        options.UseSqlServer(connectionString), ServiceLifetime.Singleton);
    ```
- **Inconvénients** :
    - Le contexte n’est pas thread-safe : risque de corruption de données et d’exceptions.
    - Ne convient pas à la plupart des scénarios.

### Quelle option choisir ?

- **Scoped** est le choix recommandé pour les applications web (ASP.NET Core), car il garantit la cohérence et la sécurité du suivi des entités pendant chaque requête.
- **Transient** peut être utilisé pour des traitements ponctuels, mais attention aux incohérences.
- **Singleton** doit être évité avec `DbContext`.

Il vaut mieux toujours utiliser le cycle de vie *scoped* pour injecter le `DbContext` dans les applications web. Cela assure la cohérence, la sécurité et la performance du suivi des entités avec Entity Framework.


## DbContextFactory 
Le `DbContextFactory` est un mécanisme d’Entity Framework Core permettant de créer des instances de `DbContext` à la demande, plutôt que de les injecter directement.  
Il est particulièrement utile dans les scénarios où :

- Un contexte est requis en dehors du cycle de vie classique (ex : tâches parallèles, services en arrière-plan, ou composants non gérés par l’injection de dépendances).
- Il est nécessaire de garantir qu’une nouvelle instance du contexte est créée à chaque utilisation, évitant ainsi les problèmes de concurrence ou de partage d’état.

Pour injecter `IDbContextFactory<TContext>` dans une application .NET (ex : ASP.NET Core), il suffit d’ajouter la ligne suivante dans la configuration des services (généralement dans `Program.cs` ou `Startup.cs`) :

```csharp
services.AddDbContextFactory<MyDbContext>(options =>
  options.UseSqlServer(connectionString));
```
  
Ajouter `AddDbContextFactory` dans la configuration des services permet d’injecter `IDbContextFactory<TContext>` dans les classes et de créer des contextes à la demande.


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
- Chaque appel à `CreateDbContext()` retourne une nouvelle instance, isolée et thread-safe.
- Idéal pour les traitements parallèles ou asynchrones.

**Résumé :**  
Le `DbContextFactory` permet de créer des contextes à la volée, assurant une gestion sûre et flexible du cycle de vie du contexte Entity Framework Core.

## DbContext et Connections

Le nombre d’instances de `DbContext` dans une application influence directement le nombre de connexions ouvertes vers la base de données :

- **Chaque `DbContext` peut ouvrir sa propre connexion** à la base, généralement lors de la première requête.
- Si tu crées beaucoup d’instances de `DbContext` (ex : injection en mode transient ou création manuelle répétée), tu risques d’ouvrir autant de connexions simultanées.
- Les connexions sont gérées par un pool (connection pooling), mais il existe une limite : si trop de contextes sont actifs en même temps, tu peux atteindre le nombre maximal de connexions autorisées par le serveur SQL.
- Cela peut entraîner des erreurs de type : “Too many connections”, ralentir l’application ou provoquer des blocages.

**Bonne pratique :**
- Utilise le cycle de vie *scoped* pour limiter le nombre de contextes vivants en même temps.
- Libère rapidement les contextes (et donc les connexions) après usage.
- Évite de créer des contextes inutiles ou de les garder trop longtemps en mémoire.

**Résumé :**  
Plus il y a d’instances de `DbContext` vivantes, plus il y a de connexions ouvertes : il faut donc gérer leur cycle de vie avec attention pour éviter la saturation du serveur de base de données.