---
title: "Les differentes versions de .NET : Framework, Core, Standard, et le .NET unifié"
tags: dotnet dotnet-core dotnet-framework dotnet-standard
---
Comment s'y retrouver dans les différentes versions de .NET : 
- .NET Framework (souvent appelé “legacy” aujourd’hui)  
- .NET Core  
- .NET (5, 6, 7, 8…) – le “.NET unifié”  
- .NET Standard (qui n’est pas un runtime, mais une spécification)
<!--more-->
# Les runtimes .NET : historique et différences

## 1. .NET Framework (“legacy”)

C’est la **plateforme historique Windows** :

- Versions principales : 1.0 → 4.8 (4.8 étant la dernière grande version).  
- **Plateforme** : uniquement Windows (WinForms, WPF, ASP.NET “classique”, services Windows…).  
- **Distribution** : installé au niveau du système (GAC, machine-wide).  
- **Support Microsoft** : maintenance et correctifs de sécurité, mais plus de nouvelles features majeures.

Caractéristiques importantes :

- Étroitement intégré à **Windows** (Registry, GDI+, COM, IIS…).  
- Beaucoup d’applications entreprises, outils internes, vieux sites ASP.NET WebForms / MVC 5 tournent dessus.  
- Écosystème très riche (librairies, intégration AD, etc.), mais :
  - pas cross-plateforme,
  - pas de nouvelles évolutions significatives,
  - performances et modèle de déploiement moins modernes que le reste de la famille.

En pratique aujourd’hui :

- On reste sur .NET Framework **uniquement** si :
  - on a une grosse base de code existante difficile à migrer,
  - on dépend fortement d’APIs Windows spécifiques (WinForms lourd, WPF avec librairies non portées, COM, etc.),
  - le coût/risque de migration est trop élevé à court terme.
- Pour **tout nouveau projet**, Microsoft recommande **.NET (Core/5+)**, pas .NET Framework.

## 2. .NET Core

C’est la première génération “moderne” :

- Versions : 1.0 → 3.1.  
- **Plateforme** : Windows, Linux, macOS.  
- **Types d’apps** : ASP.NET Core, services console, microservices, etc.  
- **Modèle de déploiement** :
  - self-contained possible (runtime livré avec l’app),
  - side-by-side (plusieurs versions .NET Core sur la même machine).

Objectifs principaux :

- Être **multiplateforme**.  
- Améliorer les **performances** (JIT, GC, Kestrel, etc.).  
- Simplifier le **déploiement** (plus dépendant de l’OS comme Framework).  
- Moderniser l’outillage (CLI `dotnet`, `csproj` simplifiés, etc.).

Statut aujourd’hui :

- .NET Core 1.x / 2.x / 3.0 sont **hors support**.
- .NET Core 3.1 a été un temps LTS, mais est aussi hors support maintenant.
- La **suite logique** de .NET Core, c’est **.NET 5, 6, 7, 8…**.  
  On n’utilise quasiment plus le terme “.NET Core” pour les nouvelles versions, juste “.NET”.

## 3. .NET 5, 6, 7, 8… (le .NET unifié)

À partir de **.NET 5**, Microsoft a unifié la plateforme et le branding :

- On dit juste **“.NET”** (ou “.NET 6”, “.NET 8”, etc.).  
- Techniquement, c’est la continuité de **.NET Core** (même runtime de base, même CLI `dotnet`), mais :
  - avec la convergence de **.NET Framework, .NET Core, Xamarin, Mono** autour d’un même “.NET”,
  - et des workloads (Web, Desktop, Mobile, Cloud, IoT…) sur la même base.

Points clés :

- **Cross-plateforme** : Windows, Linux, macOS (et plus, via MAUI/Xamarin pour mobile).  
- **Performances élevées**, gros investissements sur :
  - le JIT (RyuJIT),
  - le GC,
  - les IO asynchrones,
  - ASP.NET Core, gRPC, minimal APIs, etc.
- **Cycle de release annuel** :
  - versions **LTS** (support long) tous les 2 ans (ex. .NET 6, .NET 8),
  - versions “courtes” (STS) entre les LTS (ex. .NET 7) avec support plus réduit.
- **Écosystème unifié** :
  - même SDK/CLI pour toutes les applis,
  - projets SDK-style avec `TargetFramework` (ex. `net8.0`).

En pratique aujourd’hui :

- Pour **tout nouveau projet**, le choix par défaut est une version récente de **.NET (LTS si possible, ex. .NET 8)**.  
- Lorsque des documentations mentionnent encore “.NET Core 6/7/8”, il s’agit en général d’un abus de langage : officiellement, on parle simplement de “.NET 6/7/8”.

# Les spécifications d’API communes

## C'est quoi ".NET Standard" ?

**.NET Standard n’est pas un runtime**. C’est :

- Une **spécification d’API commune** que plusieurs runtimes implémentent :
  - .NET Framework,
  - .NET Core,
  - Mono / Xamarin,
  - et dans une certaine mesure .NET 5+ (compatibilité ascendante).
- L’objectif : écrire une **librairie réutilisable** qui tourne sur **plusieurs runtimes .NET**.

`.NET Standard` dit : “voici l’ensemble minimal d’API BCL (Base Class Library) que tous les runtimes qui se réclament de la version X doivent implémenter”. Une librairie compilée pour `netstandard2.0` peut **en principe** être utilisée :
  - par une appli `.NET Framework 4.6.1+`,  
  - par une appli `.NET Core 2.0+`,  
  - par une appli `.NET 5+`.

## Les versions importantes

- **.NET Standard 1.x** : très limité, peu d’API, compatible avec beaucoup de runtimes mais pas très pratique.  
- **.NET Standard 2.0** :
  - gros saut : ~20k APIs,
  - compatible avec .NET Framework 4.6.1+, .NET Core 2.0+, Mono…
  - c’est la **version la plus utilisée** pour les libs cross-plateforme.
- **.NET Standard 2.1** :
  - plus riche, mais **non supporté par .NET Framework**,
  - supporté par .NET Core 3.0+ et .NET 5+,
  - moins utilisé car limite la compatibilité avec les projets legacy.

## Les principales API disponibles en .NET Standard 2.0

.NET Standard 2.0 couvre une grande partie de la BCL (Base Class Library). Concrètement, on y retrouve la majorité des briques “génériques” dont une librairie a besoin :

- **Base du langage et collections**  
  - `System`, `System.Object`, `System.String`, `System.DateTime`, `System.Guid`, etc.  
  - Collections génériques : `System.Collections.Generic` (`List<T>`, `Dictionary<TKey,TValue>`, `HashSet<T>`, `Queue<T>`, `Stack<T>`, etc.).  
  - Collections immuables et concurrentes (`System.Collections.Immutable`, `System.Collections.Concurrent`).

- **LINQ et manipulation de données**  
  - `System.Linq` (opérateurs LINQ standard : `Select`, `Where`, `OrderBy`, `GroupBy`, `Join`, etc.).  
  - `System.Linq.Expressions` (arbres d’expressions, utiles pour ORMs, moteurs de règles, etc.).

- **Fichiers, flux et IO**  
  - `System.IO` (`File`, `Directory`, `FileStream`, `StreamReader`, `StreamWriter`, etc.).  
  - `System.IO.Compression` (ZIP, GZip, Deflate…).

- **Réseau et HTTP**  
  - `System.Net`, `System.Net.Sockets` (sockets TCP/UDP, DNS, etc.).  
  - `System.Net.Http` (`HttpClient`, `HttpRequestMessage`, `HttpResponseMessage`, `HttpContent`, etc.).

- **Sérialisation et formats**  
  - `System.Xml`, `System.Xml.Linq` (LINQ to XML).  
  - `System.Runtime.Serialization` (DataContract, etc.).  
  - Les libs JSON modernes (ex. `System.Text.Json`, `Newtonsoft.Json`) sont disponibles sous forme de packages NuGet ciblant netstandard2.0.

- **Sécurité / cryptographie**  
  - `System.Security.Cryptography` (hash, HMAC, RSA, AES, etc.).

- **Multithreading et tâches asynchrones**  
  - `System.Threading`, `System.Threading.Tasks` (`Task`, `Task<T>`, `CancellationToken`, `SemaphoreSlim`, etc.).  
  - Timers (`System.Timers.Timer`, `System.Threading.Timer`).

- **Reflection, runtime, diagnostics**  
  - `System.Reflection` (inspection de types, assemblies, attributs…).  
  - `System.Runtime` (types de base du runtime).  
  - `System.Diagnostics` (`Debug`, `Trace`, `Process`, compteurs de performance…).

Ce qui **n’est pas** dans .NET Standard 2.0 (ou pas directement) :

- Les **frameworks UI** spécifiques : WinForms, WPF, UWP, MAUI, Xamarin.Forms…  
- L’infrastructure **ASP.NET / ASP.NET Core** (middleware, contrôleurs MVC, Razor, etc.) : elle vit dans des packages ciblant différents runtimes, pas dans le “cœur” de .NET Standard.  
- Les **ORMS** comme Entity Framework / EF Core : disponibles via NuGet, souvent multi-ciblage (`netstandard2.0`, `net6.0`, etc.), mais ce n’est pas du pur .NET Standard.

En résumé, une librairie `netstandard2.0` a accès à l’essentiel des API de base nécessaires pour :

- manipuler des types métier, des collections et des dates,  
- faire des IO (fichiers, flux),  
- parler HTTP,  
- sérialiser / désérialiser des données,  
- gérer l’asynchronisme et la concurrence,  
- introspecter des types via la reflection.  

C’est précisément ce socle riche qui en a fait la cible privilégiée pour les bibliothèques NuGet “génériques” compatibles à la fois avec .NET Framework, .NET Core et .NET moderne.

Statut maintenant :

- Avec l’arrivée de **.NET 5+**, Microsoft pousse plutôt vers :
  - cibler directement `net6.0`, `net8.0`, etc.,
  - et utiliser **multi-targeting** si besoin (`netstandard2.0` + `net8.0`, par exemple).
- .NET Standard reste utile si :
- une bibliothèque NuGet doit fonctionner sur **.NET Framework + .NET Core + .NET moderne**,  
- l’environnement n’est pas encore prêt à abandonner .NET Framework.

# Comment positionner “legacy / core / standard” dans ta tête

## En résumé rapide

- **.NET Framework**  
  - Historique, Windows-only, considéré comme **legacy** pour les nouveaux développements.  
  - À garder pour les applis Windows existantes et difficiles à migrer.

- **.NET Core**  
  - Ancien nom des versions 1.0 → 3.1 du .NET moderne.  
  - Cross-plateforme, très performant, mais désormais remplacé par **.NET 5+**.  
  - Quand quelqu’un dit “app .NET Core”, il veut souvent dire “app ASP.NET Core tournant sur le runtime moderne (.NET 6/7/8)”.

- **.NET (5, 6, 7, 8…)**  
  - **Présent et futur de la plateforme**.  
  - Cross-plateforme, unifié, fortement optimisé.  
  - Choix par défaut pour tout nouveau projet.

- **.NET Standard**  
  - Pas un runtime, une **couche de compatibilité d’API**.  
  - Sert à écrire des **libs réutilisables** qui fonctionnent sur plusieurs runtimes.  
  - Moins central pour les nouvelles applis 100% .NET 6/8+, mais encore utile pour les libs NuGet “grand public” qui doivent rester compatibles avec .NET Framework.

## Quand choisir quoi (en pratique)

- **Nouvelle application web / API / microservice**  
  → Cibler **.NET 8 (LTS)** ou la version LTS actuelle (`net8.0`), en ASP.NET Core.  
  → Pas de .NET Standard nécessaire, sauf si tu publies une lib réutilisable.

- **Nouvelle application desktop cross-plateforme**  
  → .NET 8 + MAUI ou Avalonia (selon le framework choisi).  

- **Librairie NuGet à destination large (legacy + moderne)**  
  → Multi-targeting :  
  - `netstandard2.0` (pour .NET Framework 4.6.1+, .NET Core 2+, etc.),  
  - + éventuellement `net8.0` pour profiter des API plus modernes quand dispo.

- **Gros existant en .NET Framework**  
  → Continuer en Framework à court terme,  
  → envisager **migration progressive** vers .NET moderne (par morceaux, via services séparés, ou via multi-targeting).

