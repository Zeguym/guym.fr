---
title: "Mesurer avec BenchmarkDotNet"
description: "BenchmarkDotNet : mesurer temps et allocations de code .NET de façon fiable, comparer des runtimes et éviter les pièges du mode Debug, du code mort et du setup."
tags: dotnet test performance benchmarkdotnet
---

« Cette version est plus rapide » : sans mesure, ce n'est qu'une intuition. Un `Stopwatch` autour d'une boucle donne des chiffres faussés par le JIT, le GC et le bruit du système. **BenchmarkDotNet** est l'outil de référence en .NET pour mesurer proprement du code : préchauffage, itérations multiples, statistiques, mesure des allocations. On l'applique ici à du code typique d'une application IoT : générer l'identifiant d'un capteur et exporter ses mesures en CSV.

<!--more-->

# 1. Pourquoi pas un Stopwatch ?

Prenons une plateforme qui collecte les mesures de capteurs répartis sur plusieurs sites. Chaque capteur est identifié par un libellé du type `SNS-0012-000042` (site sur 4 chiffres, numéro de capteur sur 6), produit par une petite méthode :

```csharp
using System.Globalization;

namespace Sensors.Domain;

public static class SensorLabel
{
    // SNS-0012-000042
    public static string Format(int site, int sensor) =>
        string.Create(CultureInfo.InvariantCulture, $"SNS-{site:D4}-{sensor:D6}");
}
```

Première tentative de mesure :

```csharp
// ❌ Mesure naïve : résultat quasiment inexploitable
var sw = Stopwatch.StartNew();
for (var i = 0; i < 1_000; i++)
    _ = SensorLabel.Format(12, i);
sw.Stop();
Console.WriteLine(sw.Elapsed);
```

Ce code mesure pêle-mêle la compilation JIT du premier appel, d'éventuelles collectes du GC, le *tiered compilation* qui recompile la méthode en cours de route, et n'importe quel processus qui tourne en parallèle sur la machine. Une seule exécution, aucune statistique : impossible de dire si un écart de 10 % est réel.

BenchmarkDotNet automatise tout ce qu'il faudrait faire à la main :

```mermaid
graph LR
    A["Build Release<br/>(projet généré)"]
    B["Pilot<br/>(nb d'itérations)"]
    C["Overhead<br/>(coût à vide)"]
    D["Warmup<br/>(JIT, caches)"]
    E["Workload<br/>(mesures réelles)"]
    F["Statistiques<br/>Mean, Error, Allocated"]

    A --> B --> C --> D --> E --> F

    style A fill:#4A90E2
    style D fill:#F5A623
    style E fill:#F5A623
    style F fill:#7ED321
```

Chaque benchmark s'exécute dans un **processus séparé**, pour éviter qu'un benchmark n'influence le suivant (JIT, état du GC, caches).

# 2. Créer le projet de benchmark

Un benchmark vit dans un **projet console dédié**, jamais dans le projet de tests unitaires :

```bash
dotnet new console -n Sensors.Benchmarks -o benchmarks/Sensors.Benchmarks
dotnet add benchmarks/Sensors.Benchmarks package BenchmarkDotNet --version 0.15.8
dotnet add benchmarks/Sensors.Benchmarks reference src/Sensors.Domain
```

Le point d'entrée délègue à `BenchmarkSwitcher`, qui permet de choisir les benchmarks en ligne de commande :

```csharp
// Program.cs
using BenchmarkDotNet.Running;

BenchmarkSwitcher.FromAssembly(typeof(Program).Assembly).Run(args);
```

Et surtout, on lance **toujours en Release** :

```bash
# Tous les benchmarks
dotnet run -c Release --project benchmarks/Sensors.Benchmarks -- --filter "*"

# Seulement ceux dont le nom contient "Label"
dotnet run -c Release --project benchmarks/Sensors.Benchmarks -- --filter "*Label*"
```

En Debug, le JIT n'optimise pas le code : les chiffres n'ont aucun sens. BenchmarkDotNet le détecte et refuse de lancer les mesures sur un assembly non optimisé.

# 3. Premier benchmark : le libellé du capteur

Le libellé `SNS-0012-000042` est généré à chaque mesure reçue : c'est un chemin chaud. On compare quatre façons de le construire :

```csharp
using System.Globalization;
using System.Text;
using BenchmarkDotNet.Attributes;

namespace Sensors.Benchmarks;

[MemoryDiagnoser] // ajoute les colonnes Gen0 et Allocated
public class SensorLabelBenchmarks
{
    // Champs et non constantes : évite que le JIT calcule le résultat à la compilation
    private int _site = 12;
    private int _sensor = 42;

    [Benchmark(Baseline = true)]
    public string Concatenation() =>
        "SNS-" + _site.ToString("D4", CultureInfo.InvariantCulture)
        + "-" + _sensor.ToString("D6", CultureInfo.InvariantCulture);

    [Benchmark]
    public string Interpolation() =>
        string.Create(CultureInfo.InvariantCulture, $"SNS-{_site:D4}-{_sensor:D6}");

    [Benchmark]
    public string Builder() =>
        new StringBuilder(15)
            .Append("SNS-")
            .Append(CultureInfo.InvariantCulture, $"{_site:D4}")
            .Append('-')
            .Append(CultureInfo.InvariantCulture, $"{_sensor:D6}")
            .ToString();

    [Benchmark]
    public string StringCreate() =>
        // Longueur connue : 4 ("SNS-") + 4 + 1 + 6 = 15 caractères
        string.Create(15, (_site, _sensor), static (span, state) =>
        {
            "SNS-".CopyTo(span);
            state._site.TryFormat(span[4..8], out _, "D4", CultureInfo.InvariantCulture);
            span[8] = '-';
            state._sensor.TryFormat(span[9..], out _, "D6", CultureInfo.InvariantCulture);
        });
}
```

Points importants :

- **Chaque méthode retourne son résultat**. BenchmarkDotNet consomme la valeur retournée, ce qui empêche le JIT de supprimer le calcul comme du code mort.
- `Baseline = true` désigne la référence : les autres méthodes sont exprimées en ratio par rapport à elle.
- `[MemoryDiagnoser]` mesure les allocations managées **par opération**.

# 4. Lire les résultats

Après quelques dizaines de secondes, BenchmarkDotNet affiche un tableau (et l'exporte dans `BenchmarkDotNet.Artifacts/results`). Voici l'allure des résultats ; **les valeurs sont indicatives** et varient selon la machine, l'OS et la version du runtime :

```text
| Method        | Mean      | Error     | StdDev    | Ratio | Gen0   | Allocated | Alloc Ratio |
|-------------- |----------:|----------:|----------:|------:|-------:|----------:|------------:|
| Concatenation |  ~30 ns   |  ~0.5 ns  |  ~0.4 ns  |  1.00 | 0.0095 |     120 B |        1.00 |
| Interpolation |  ~25 ns   |  ~0.3 ns  |  ~0.3 ns  | ~0.85 | 0.0044 |      56 B |        0.47 |
| Builder       |  ~35 ns   |  ~0.6 ns  |  ~0.5 ns  | ~1.15 | 0.0120 |    ~150 B |       ~1.25 |
| StringCreate  |  ~15 ns   |  ~0.2 ns  |  ~0.2 ns  | ~0.50 | 0.0044 |      56 B |        0.47 |
```

| Colonne | Signification |
|---|---|
| **Mean** | Moyenne arithmétique du temps d'**une** opération |
| **Error** | Demi-largeur de l'intervalle de confiance à 99,9 % : la vraie moyenne est dans `Mean ± Error` |
| **StdDev** | Écart-type des mesures : une valeur élevée signale du bruit |
| **Ratio** | Temps rapporté à la baseline (< 1 = plus rapide) |
| **Gen0** | Nombre de collectes de génération 0 pour 1 000 opérations |
| **Allocated** | Mémoire managée allouée par opération |
| **Alloc Ratio** | Allocations rapportées à la baseline |

Ce qu'on en retient :

- La **concaténation** alloue deux chaînes intermédiaires (`ToString("D4")`, `ToString("D6")`) en plus du résultat.
- L'**interpolation** est déjà très bonne depuis .NET 6 : le compilateur utilise `DefaultInterpolatedStringHandler`, qui formate dans un buffer emprunté et n'alloue que la chaîne finale.
- `StringBuilder` n'apporte rien sur une petite chaîne : on paie l'objet et son buffer.
- `string.Create` écrit directement dans la mémoire de la chaîne finale : c'est le plus rapide, au prix d'un code moins lisible.

Conclusion pragmatique : l'interpolation est le bon choix par défaut ; `string.Create` se justifie sur un chemin très chaud, **mesuré** comme tel.

Pour comprendre pourquoi ces allocations comptent, voir l'article sur [la pile et le tas](/2025/11/22/dotnet-basics-stack-heap.html) : chaque objet alloué sur le tas finit par coûter du temps au GC.

# 5. Faire varier les entrées avec [Params]

Un résultat dépend souvent de la taille des données. Les mesures d'un capteur sont régulièrement exportées en CSV, une ligne par mesure : on compare concaténation et `StringBuilder` pour 10, 100 et 1 000 mesures. Une mesure est un simple record :

```csharp
namespace Sensors.Domain;

// Horodatage en secondes Unix, valeur en degrés Celsius
public sealed record SensorReading(string SensorId, long UnixTime, double Value);
```

```csharp
using System.Globalization;
using System.Text;
using BenchmarkDotNet.Attributes;
using Sensors.Domain;

namespace Sensors.Benchmarks;

[MemoryDiagnoser]
public class CsvExportBenchmarks
{
    [Params(10, 100, 1_000)] // chaque benchmark est exécuté pour chaque valeur
    public int ReadingCount { get; set; }

    private SensorReading[] _readings = [];

    // Exécuté une fois par combinaison de paramètres, hors mesure
    [GlobalSetup]
    public void Setup() =>
        _readings = Enumerable.Range(1, ReadingCount)
            .Select(i => new SensorReading("SNS-0012-000042", 1_775_000_000 + i * 60L, 18.0 + i % 70 / 10.0))
            .ToArray();

    [Benchmark(Baseline = true)]
    public string Concatenation()
    {
        var csv = "";
        foreach (var r in _readings)
            csv += string.Create(CultureInfo.InvariantCulture,
                $"{r.SensorId};{r.UnixTime};{r.Value}\n");
        return csv;
    }

    [Benchmark]
    public string Builder()
    {
        var sb = new StringBuilder();
        foreach (var r in _readings)
            sb.Append(CultureInfo.InvariantCulture, $"{r.SensorId};{r.UnixTime};{r.Value}").Append('\n');
        return sb.ToString();
    }
}
```

Résultats indicatifs :

```text
| Method        | ReadingCount | Mean        | Ratio | Allocated  | Alloc Ratio |
|-------------- |------------- |------------:|------:|-----------:|------------:|
| Concatenation | 10           |    ~1 µs    |  1.00 |    ~5 KB   |        1.00 |
| Builder       | 10           |    ~0.8 µs  | ~0.80 |    ~2.5 KB |       ~0.50 |
| Concatenation | 100          |   ~20 µs    |  1.00 |  ~350 KB   |        1.00 |
| Builder       | 100          |    ~8 µs    | ~0.40 |   ~25 KB   |       ~0.07 |
| Concatenation | 1000         |  ~2 000 µs  |  1.00 |   ~30 MB   |        1.00 |
| Builder       | 1000         |   ~80 µs    | ~0.04 |  ~250 KB   |      ~0.008 |
```

Le comportement **quadratique** de la concaténation en boucle saute aux yeux : chaque `+=` recopie toute la chaîne existante. Avec 10 mesures, la différence est négligeable ; avec 1 000, elle est de plusieurs ordres de grandeur. C'est exactement ce que `[Params]` permet de révéler : **un benchmark sur une seule taille peut mentir**.

D'autres attributs complètent `[Params]` :

| Attribut | Usage |
|---|---|
| `[Params(...)]` | Valeurs constantes d'une propriété ou d'un champ public |
| `[ParamsSource(nameof(X))]` | Valeurs calculées par une propriété ou une méthode |
| `[Arguments(...)]` | Arguments passés directement à la méthode benchmark |
| `[GlobalSetup]` / `[GlobalCleanup]` | Préparation et nettoyage, hors mesure |

# 6. LINQ ou boucle ?

Autre question classique : calculer la moyenne des mesures d'un capteur. On ajoute deux méthodes à la classe `CsvExportBenchmarks` (ou à une classe dédiée avec le même `[GlobalSetup]`) :

```csharp
[Benchmark]
public double AverageLinq() => _readings.Average(r => r.Value);

[Benchmark]
public double AverageLoop()
{
    var sum = 0.0;
    foreach (var r in _readings)
        sum += r.Value;
    return sum / _readings.Length;
}
```

La boucle évite l'appel du délégué à chaque élément et sera généralement un peu plus rapide, avec des allocations nulles ou marginales dans les deux cas (le lambda ne capture rien, il est mis en cache par le compilateur). L'écart reste souvent faible devant le coût d'un accès réseau ou base de données : c'est le genre de micro-optimisation qu'il ne faut appliquer qu'**après mesure**, sur un chemin chaud. Les [bonnes pratiques LINQ](/2026/02/08/linq-best-practices.html) détaillent les cas où LINQ coûte réellement cher (allocations cachées, énumérations multiples).

# 7. Les pièges classiques

## Mesurer en Debug ou avec un débogueur attaché

Toujours `-c Release`, sans débogueur, en ligne de commande. Lancer les benchmarks depuis l'IDE avec F5 fausse les résultats.

## Le code mort

```csharp
// ❌ Le résultat n'est pas utilisé : le JIT peut supprimer le calcul
[Benchmark]
public void DeadCode() => _ = SensorLabel.Format(_site, _sensor);

// ❌ Constantes : le résultat peut être calculé à la compilation
[Benchmark]
public double Folded() => 21.5 * 1.8 + 32;

// ✅ On retourne le résultat, calculé à partir de champs
[Benchmark]
public string Alive() => SensorLabel.Format(_site, _sensor);
```

## Ne pas mesurer en isolation

- **Machine au calme** : fermer navigateur, IDE, Teams ; brancher un portable sur secteur (les modes économie d'énergie brident le CPU).
- **Pas de benchmark en CI partagée** pour en tirer des conclusions absolues : les machines virtuelles mutualisées sont bruyantes. On peut en revanche y comparer des ratios.
- **Comparer ce qui est comparable** : même runtime, même machine, même exécution. Un `[Params]` ou plusieurs runtimes dans une même session valent mieux que deux sessions à des jours d'écart.

## Inclure le setup dans la mesure

```csharp
// ❌ La création des données est mesurée avec le reste
[Benchmark]
public string Builder()
{
    var readings = Enumerable.Range(1, 1_000)
        .Select(i => new SensorReading("SNS-0012-000042", 1_775_000_000 + i * 60L, 21.5))
        .ToArray();
    // ... construction du CSV
}
// ✅ Préparation dans [GlobalSetup], comme dans CsvExportBenchmarks
```

## Généraliser un micro-benchmark

Un benchmark mesure un morceau de code isolé, avec des caches chauds. En production, ce code partage le CPU, la mémoire et le GC avec tout le reste. Un gain de 50 % sur 30 ns n'a aucun impact visible sur un endpoint qui attend 20 ms la base de données. Le micro-benchmark valide une hypothèse ; le **profilage** et la télémétrie en production disent où se trouve le vrai problème.

## Comparer plusieurs runtimes

Pour vérifier le gain d'une migration, on cible plusieurs frameworks dans le `.csproj` (`<TargetFrameworks>net9.0;net10.0</TargetFrameworks>`) et on demande les deux runtimes :

```bash
dotnet run -c Release -f net10.0 --project benchmarks/Sensors.Benchmarks -- --filter "*Csv*" --runtimes net9.0 net10.0
```

# Résumé

| Élément | Rôle |
|---|---|
| Projet console dédié + `dotnet run -c Release` | Mesures sur du code optimisé, hors tests unitaires |
| `BenchmarkSwitcher` + `--filter` | Choisir les benchmarks à lancer |
| `[Benchmark]` / `Baseline = true` | Méthode mesurée / référence des ratios |
| `[MemoryDiagnoser]` | Colonnes `Gen0` et `Allocated` |
| `[Params]` / `[GlobalSetup]` | Faire varier les entrées, préparer hors mesure |
| **Mean ± Error** | Temps moyen et intervalle de confiance à 99,9 % |
| **Allocated** | Mémoire allouée par opération |
| Retourner le résultat, utiliser des champs | Éviter code mort et constant folding |
| Machine au calme, même session | Mesurer en isolation, comparer ce qui est comparable |

Là où les tests disent si le code est **correct**, BenchmarkDotNet dit **combien il coûte**, en temps comme en mémoire. Sur notre plateforme de capteurs, il a confirmé deux intuitions et en a chiffré une troisième : l'interpolation suffit pour un libellé, `string.Create` ne se justifie que sur un chemin très chaud, et la concaténation en boucle devient vite catastrophique. Pour la partie correction, voir [xUnit v3 : organiser ses tests](/2026/04/04/tests-xunit-v3.html).
