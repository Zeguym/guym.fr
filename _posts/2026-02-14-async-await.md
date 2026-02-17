---
title: "Async/Await en C# : comprendre la programmation asynchrone"
tags: dotnet basics async await csharp
---

La programmation asynchrone avec `async`/`await` est au cœur du développement .NET moderne. Elle permet de libérer des threads pendant les opérations d'entrée/sortie (réseau, fichiers, base de données) au lieu de les bloquer. Mais derrière cette syntaxe simple se cachent une machine à états, un `SynchronizationContext`, et de nombreux pièges. Cet article détaille le fonctionnement réel d'`async`/`await`, de la théorie aux bonnes pratiques.

<!--more-->

# Pourquoi l'asynchrone ?

## Le problème : le thread bloqué

Dans un serveur web, chaque requête HTTP est traitée par un thread du pool. Si ce thread effectue un appel réseau **synchrone**, il reste bloqué en attente de la réponse :

```csharp
// ❌ Synchrone : le thread est bloqué pendant toute la durée de l'appel HTTP
public IActionResult GetData()
{
    var client = new HttpClient();
    var response = client.Send(new HttpRequestMessage(HttpMethod.Get, "https://api.example.com/data"));
    var content = new StreamReader(response.Content.ReadAsStream()).ReadToEnd();
    return Ok(content);
}
```

Avec 100 requêtes simultanées et un pool de 100 threads, le serveur est **saturé** : plus aucun thread disponible pour traiter de nouvelles requêtes, alors que chaque thread ne fait que **attendre**.

## La solution : libérer le thread pendant l'attente

```csharp
// ✅ Asynchrone : le thread est libéré pendant l'appel HTTP
public async Task<IActionResult> GetData()
{
    var client = new HttpClient();
    var response = await client.GetAsync("https://api.example.com/data");
    var content = await response.Content.ReadAsStringAsync();
    return Ok(content);
}
```

Avec `await`, le thread est **rendu au pool** pendant l'attente réseau. Il peut traiter d'autres requêtes. Quand la réponse arrive, un thread du pool reprend l'exécution là où elle s'était arrêtée.

## Ce que l'asynchrone n'est PAS

- **Ce n'est PAS du parallélisme** : `async`/`await` ne crée pas de thread supplémentaire. Il libère le thread courant.
- **Ce n'est PAS plus rapide** pour une seule opération : l'appel HTTP prend toujours le même temps. Le gain est en **scalabilité** (plus de requêtes simultanées avec moins de threads).
- **Ce n'est PAS utile pour du calcul CPU** : si le travail est du calcul pur (boucle, algorithme), `await` n'apporte rien — le thread est occupé à calculer, pas à attendre.

# `Task` et `Task<T>` : la promesse d'un résultat

## `Task` : une opération sans résultat

```csharp
public async Task EnvoyerEmailAsync(string destinataire, string message)
{
    await smtpClient.SendMailAsync(destinataire, message);
    // Pas de valeur de retour
}
```

## `Task<T>` : une opération avec résultat

```csharp
public async Task<Client> GetClientAsync(int id)
{
    var client = await dbContext.Clients.FindAsync(id);
    return client; // retourne un Client, encapsulé dans Task<Client>
}
```

Un `Task` représente une **opération en cours** (ou déjà terminée). C'est l'équivalent d'une "promesse" (Promise en JavaScript). On peut :
- **Attendre** son résultat avec `await`.
- Vérifier son état : `IsCompleted`, `IsFaulted`, `IsCanceled`.
- Accéder au résultat (si terminé) via `.Result` — mais **attention aux deadlocks** (voir plus bas).

## `ValueTask<T>` : l'alternative légère

`ValueTask<T>` évite l'allocation d'un objet `Task` sur le tas quand le résultat est **souvent déjà disponible** (cache, valeur par défaut) :

```csharp
// Si le cache contient la valeur, pas besoin d'allouer un Task
public ValueTask<Client> GetClientAsync(int id)
{
    if (_cache.TryGetValue(id, out var client))
        return new ValueTask<Client>(client); // pas d'allocation

    return new ValueTask<Client>(GetClientFromDbAsync(id)); // allocation Task si nécessaire
}

private async Task<Client> GetClientFromDbAsync(int id)
{
    return await dbContext.Clients.FindAsync(id);
}
```

**Règle** : utiliser `ValueTask<T>` quand la méthode retourne souvent un résultat **synchrone** (cache hit, valeur par défaut). Sinon, `Task<T>` suffit.

# Ce que le compilateur génère

## La machine à états

Quand le compilateur rencontre `async`/`await`, il transforme la méthode en une **machine à états** (`IAsyncStateMachine`). Chaque `await` correspond à un **point de suspension**.

```csharp
// Ce qu'on écrit
public async Task<string> GetDataAsync()
{
    var response = await httpClient.GetAsync("https://api.example.com");
    var content = await response.Content.ReadAsStringAsync();
    return content.ToUpper();
}
```

Le compilateur génère (version très simplifiée) :

```csharp
// Ce que le compilateur produit
private struct GetDataAsyncStateMachine : IAsyncStateMachine
{
    public int _state;  // -1 = début, 0 = après 1er await, 1 = après 2ème await
    public AsyncTaskMethodBuilder<string> _builder;
    public HttpClient httpClient;

    // Variables locales "promues" en champs
    private HttpResponseMessage _response;
    private string _content;

    // Awaiters pour chaque await
    private TaskAwaiter<HttpResponseMessage> _awaiter1;
    private TaskAwaiter<string> _awaiter2;

    public void MoveNext()
    {
        try
        {
            switch (_state)
            {
                case -1: // début de la méthode
                    _awaiter1 = httpClient.GetAsync("https://api.example.com").GetAwaiter();
                    if (!_awaiter1.IsCompleted)
                    {
                        _state = 0;
                        _builder.AwaitUnsafeOnCompleted(ref _awaiter1, ref this);
                        return; // le thread est libéré ICI
                    }
                    goto case 0;

                case 0: // reprise après le 1er await
                    _response = _awaiter1.GetResult();
                    _awaiter2 = _response.Content.ReadAsStringAsync().GetAwaiter();
                    if (!_awaiter2.IsCompleted)
                    {
                        _state = 1;
                        _builder.AwaitUnsafeOnCompleted(ref _awaiter2, ref this);
                        return; // le thread est libéré ICI
                    }
                    goto case 1;

                case 1: // reprise après le 2ème await
                    _content = _awaiter2.GetResult();
                    _builder.SetResult(_content.ToUpper());
                    return;
            }
        }
        catch (Exception ex)
        {
            _builder.SetException(ex);
        }
    }
}
```

### Points clés

1. **Les variables locales** (`response`, `content`) deviennent des **champs** de la struct, car elles doivent survivre entre les reprises.
2. **`IsCompleted`** est vérifié d'abord : si le `Task` est déjà terminé, on continue **sans suspendre** (optimisation synchrone).
3. **`return`** dans le `MoveNext()` libère le thread. Quand l'opération asynchrone se termine, `MoveNext()` est rappelé avec le bon `_state`.
4. **Les exceptions** sont capturées et stockées dans le `Task` via `SetException`.

# Le `SynchronizationContext`

## Le problème de la reprise

Quand un `await` se termine, sur **quel thread** reprend l'exécution ? C'est le rôle du `SynchronizationContext`.

| Environnement | `SynchronizationContext` | Reprise sur |
|---|---|---|
| ASP.NET Core | Aucun (`null`) | N'importe quel thread du pool |
| WPF / WinForms | UI context | Le thread UI |
| Application console | Aucun (`null`) | N'importe quel thread du pool |
| ASP.NET classique (.NET Framework) | `AspNetSynchronizationContext` | Le même thread de requête |

## Pourquoi c'est important

En **WPF/WinForms**, après un `await`, l'exécution reprend automatiquement sur le **thread UI** (pour pouvoir mettre à jour les contrôles) :

```csharp
// WPF : après await, on est de retour sur le thread UI
private async void Button_Click(object sender, RoutedEventArgs e)
{
    var data = await httpClient.GetStringAsync("https://api.example.com");
    TextBlock.Text = data; // ✅ OK : on est sur le thread UI
}
```

En **ASP.NET Core**, il n'y a pas de `SynchronizationContext`, donc la reprise se fait sur n'importe quel thread du pool — ce qui est plus performant.

## `ConfigureAwait(false)` 

`ConfigureAwait(false)` dit : "je n'ai pas besoin de revenir sur le contexte d'origine".

```csharp
// Dans du code de bibliothèque (pas de UI, pas de contexte HTTP)
public async Task<string> GetDataAsync()
{
    var response = await httpClient.GetAsync(url).ConfigureAwait(false);
    // La reprise se fait sur n'importe quel thread du pool
    var content = await response.Content.ReadAsStringAsync().ConfigureAwait(false);
    return content;
}
```

**Règles** :
- **Code applicatif ASP.NET Core** : `ConfigureAwait(false)` est inutile (pas de `SynchronizationContext`), mais ne fait pas de mal.
- **Bibliothèques** : toujours mettre `ConfigureAwait(false)` pour éviter les deadlocks quand la bibliothèque est utilisée dans un contexte UI ou ASP.NET classique.
- **Code UI (WPF/WinForms)** : ne pas mettre `ConfigureAwait(false)` si on accède à des contrôles UI après le `await`.

# Pièges courants

## 1. `async void` — à éviter absolument

```csharp
// ❌ async void : les exceptions ne sont pas capturables
public async void EnvoyerNotification()
{
    await httpClient.PostAsync(url, content);
    // Si une exception est levée, elle crashe l'application
    // car personne ne peut await cette méthode (pas de Task)
}

// ✅ async Task : les exceptions sont propagées via le Task
public async Task EnvoyerNotificationAsync()
{
    await httpClient.PostAsync(url, content);
}
```

**Exception unique** : les event handlers UI (WPF, WinForms) **imposent** `async void` car la signature de l'événement est `void`.

```csharp
// Seul cas acceptable pour async void
private async void Button_Click(object sender, RoutedEventArgs e)
{
    try
    {
        await DoWorkAsync();
    }
    catch (Exception ex)
    {
        // Toujours un try-catch dans un async void !
        MessageBox.Show(ex.Message);
    }
}
```

## 2. Deadlock avec `.Result` ou `.Wait()`

```csharp
// ❌ DEADLOCK en WPF / ASP.NET classique
public void GetData()
{
    var data = GetDataAsync().Result; // bloque le thread UI
    // GetDataAsync attend de revenir sur le thread UI (SynchronizationContext)
    // Mais le thread UI est bloqué par .Result → DEADLOCK
}

public async Task<string> GetDataAsync()
{
    var result = await httpClient.GetStringAsync(url);
    // await veut revenir sur le thread UI, mais il est bloqué
    return result;
}
```

Le scénario du deadlock :
1. `.Result` bloque le thread courant (UI) en attendant le `Task`.
2. Le `await` dans `GetDataAsync` veut reprendre sur le thread UI (via `SynchronizationContext`).
3. Le thread UI est bloqué → **deadlock**.

**Solutions** :
```csharp
// ✅ Solution 1 : utiliser await partout (async all the way)
public async Task<string> GetData()
{
    return await GetDataAsync();
}

// ✅ Solution 2 : ConfigureAwait(false) dans la méthode appelée
public async Task<string> GetDataAsync()
{
    var result = await httpClient.GetStringAsync(url).ConfigureAwait(false);
    return result;
}
```

## 3. `async` inutile (élision d'await)

Quand une méthode ne fait que **transmettre** un `Task` sans rien faire après, `async`/`await` est superflu :

```csharp
// ❌ async/await inutile : crée une machine à états pour rien
public async Task<Client> GetClientAsync(int id)
{
    return await repository.GetByIdAsync(id);
}

// ✅ Retourner directement le Task
public Task<Client> GetClientAsync(int id)
{
    return repository.GetByIdAsync(id);
}
```

# Documentation

## Attention concernant l'élision async/await

Ce commentaire avertit que **l'élision de `async`** (c'est-à-dire la suppression du mot-clé `async` d'une méthode qui retourne directement une `Task`) modifie le comportement du code de deux façons importantes :

### 1. **Sémantique des exceptions**
- Avec `async` : les exceptions sont capturées et enveloppées dans la `Task` retournée
- Sans `async` : les exceptions sont levées directement lors de l'appel

### 2. **Stack trace**
- Avec `async` : la pile d'appels peut être modifiée/raccourcie par le compilateur
- Sans `async` : la pile d'appels originale est préservée

### Recommandation

Conserver le mot-clé `async` si la méthode contient de la validation ou de la logique **avant le `return`**, car cela garantit que :
- Toutes les exceptions sont gérées de manière cohérente
- La stack trace reste complète et utile pour le déboggage
**Attention** : l'élision change la sémantique des exceptions et du stack trace. Si la méthode contient de la validation avant le `return`, garder `async` :

```csharp
// ✅ Garder async car il y a de la logique avant le await
public async Task<Client> GetClientAsync(int id)
{
    if (id <= 0) throw new ArgumentException("Id invalide");
    return await repository.GetByIdAsync(id);
    // Sans async, l'exception serait levée à l'appel, pas encapsulée dans le Task
}
```

## 4. Oublier d'`await` un `Task`

```csharp
// ❌ Le Task est créé mais jamais attendu : fire-and-forget silencieux
public async Task TraiterCommandeAsync(Commande cmd)
{
    EnvoyerConfirmationAsync(cmd.Email); // ⚠️ pas de await !
    // L'exécution continue immédiatement
    // Si EnvoyerConfirmationAsync lève une exception, elle est perdue
}

// ✅ Toujours await les Task
public async Task TraiterCommandeAsync(Commande cmd)
{
    await EnvoyerConfirmationAsync(cmd.Email);
}
```

Le compilateur émet un **warning CS4014** pour ce cas — ne pas l'ignorer.

## 5. Lancer des tâches en séquentiel au lieu de parallèle

```csharp
// ❌ Séquentiel : 3 secondes au total (1 + 1 + 1)
var client = await GetClientAsync(id);          // attend 1s
var commandes = await GetCommandesAsync(id);    // attend 1s
var factures = await GetFacturesAsync(id);      // attend 1s

// ✅ Parallèle : ~1 seconde au total (les 3 en même temps)
var clientTask = GetClientAsync(id);
var commandesTask = GetCommandesAsync(id);
var facturesTask = GetFacturesAsync(id);

await Task.WhenAll(clientTask, commandesTask, facturesTask);

var client = clientTask.Result;      // déjà terminé, pas de blocage
var commandes = commandesTask.Result;
var factures = facturesTask.Result;

// OU en C# plus concis :
var (client, commandes, factures) = (
    await clientTask,
    await commandesTask,
    await facturesTask
);
```

`Task.WhenAll` lance toutes les opérations en parallèle et attend qu'elles soient **toutes** terminées.

# Patterns utiles

## Timeout avec `CancellationToken`

```csharp
public async Task<string> GetDataAvecTimeoutAsync(CancellationToken cancellationToken = default)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
    cts.CancelAfter(TimeSpan.FromSeconds(5)); // timeout de 5 secondes

    try
    {
        return await httpClient.GetStringAsync(url, cts.Token);
    }
    catch (OperationCanceledException) when (!cancellationToken.IsCancellationRequested)
    {
        throw new TimeoutException("L'appel a dépassé le délai de 5 secondes.");
    }
}
```

## Retry simple

```csharp
public async Task<T> AvecRetryAsync<T>(Func<Task<T>> operation, int maxRetries = 3)
{
    for (int i = 0; i < maxRetries; i++)
    {
        try
        {
            return await operation();
        }
        catch (HttpRequestException) when (i < maxRetries - 1)
        {
            await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, i))); // backoff exponentiel
        }
    }
    throw new InvalidOperationException("Ne devrait pas arriver");
}

// Utilisation
var data = await AvecRetryAsync(() => httpClient.GetStringAsync(url));
```

## `IAsyncEnumerable<T>` — streaming asynchrone

Introduit en C# 8, `IAsyncEnumerable<T>` permet de produire des éléments **un par un de manière asynchrone** :

```csharp
// Producteur : chaque élément est produit de façon asynchrone
public async IAsyncEnumerable<Client> GetClientsEnStreamAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    var page = 0;
    while (true)
    {
        var batch = await dbContext.Clients
            .OrderBy(c => c.Id)
            .Skip(page * 100)
            .Take(100)
            .ToListAsync(ct);

        if (batch.Count == 0) yield break;

        foreach (var client in batch)
            yield return client;

        page++;
    }
}

// Consommateur : itère de manière asynchrone
await foreach (var client in GetClientsEnStreamAsync())
{
    Console.WriteLine(client.Nom);
}
```

## `Channel<T>` — producteur/consommateur asynchrone

```csharp
var channel = Channel.CreateBounded<Commande>(100);

// Producteur
_ = Task.Run(async () =>
{
    await foreach (var cmd in GetCommandesEnStreamAsync())
    {
        await channel.Writer.WriteAsync(cmd);
    }
    channel.Writer.Complete();
});

// Consommateur
await foreach (var cmd in channel.Reader.ReadAllAsync())
{
    await TraiterCommandeAsync(cmd);
}
```

# `Task.Run` : quand et quand ne pas l'utiliser

## Ce que `Task.Run` fait vraiment

`Task.Run` planifie du travail sur le **ThreadPool**. C'est utile uniquement pour décharger du **calcul CPU** du thread courant.

```csharp
// ✅ Décharger un calcul lourd du thread UI
private async void CalculerButton_Click(object sender, RoutedEventArgs e)
{
    var resultat = await Task.Run(() => CalculComplexe(donnees));
    ResultatTextBlock.Text = resultat.ToString(); // retour sur le thread UI
}
```

## Quand ne PAS utiliser `Task.Run`

```csharp
// ❌ Ne pas wrapper un appel déjà asynchrone dans Task.Run
public Task<string> GetDataAsync()
{
    return Task.Run(async () =>
    {
        return await httpClient.GetStringAsync(url);
    });
}
// httpClient.GetStringAsync est déjà asynchrone, Task.Run gaspille un thread

// ✅ Appeler directement
public Task<string> GetDataAsync()
{
    return httpClient.GetStringAsync(url);
}
```

**Règle** : `Task.Run` pour le **calcul CPU** en contexte UI. Jamais dans du code serveur (ASP.NET Core), où les threads sont précieux.

# Résumé

| Concept | À retenir |
|---|---|
| **`async`/`await`** | Libère le thread pendant les opérations I/O, ne crée pas de thread |
| **`Task<T>`** | Représente une opération en cours avec un résultat futur |
| **`ValueTask<T>`** | Alternative légère quand le résultat est souvent déjà disponible |
| **Machine à états** | Le compilateur transforme la méthode en `IAsyncStateMachine` avec un `switch` sur `_state` |
| **`SynchronizationContext`** | Détermine sur quel thread reprendre après un `await` |
| **`ConfigureAwait(false)`** | Indispensable dans les bibliothèques, inutile en ASP.NET Core |
| **`async void`** | À éviter sauf pour les event handlers UI — les exceptions sont non capturables |
| **`.Result` / `.Wait()`** | Risque de deadlock — préférer `await` partout |
| **`Task.WhenAll`** | Paralléliser des opérations indépendantes |
| **`CancellationToken`** | Toujours propager pour permettre l'annulation |
| **`IAsyncEnumerable<T>`** | Streaming asynchrone élément par élément |
| **`Task.Run`** | Uniquement pour décharger du calcul CPU, jamais pour wrapper de l'async |