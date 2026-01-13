---
title: "Cibler plusieurs frameworks .NET dans un même projet"
tags: dotnet nuget multi-targeting multi-framework
---

Lorsqu’une bibliothèque doit être utilisable par plusieurs types d’applications (.NET Framework, .NET Core, .NET moderne, etc.), il est possible de **cibler plusieurs frameworks .NET dans un même projet**. Cette technique est appelée *multi‑ciblage* (*multi‑targeting*).

L’idée est simple : un seul projet `.csproj`, plusieurs frameworks cibles, et le SDK .NET génère un assembly par framework.

<!--more-->

# Principe du multi‑ciblage

Dans un projet SDK‑style, au lieu de définir une propriété `TargetFramework`, on utilise `TargetFrameworks` (au pluriel) avec une liste de **TFM** (*Target Framework Monikers*), séparés par des `;`.

Exemple : une bibliothèque qui doit fonctionner à la fois sur .NET Standard 2.0 et sur .NET 8 :

```xml
<Project Sdk="Microsoft.NET.Sdk">
	<PropertyGroup>
		<!-- Deux frameworks ciblés -->
		<TargetFrameworks>netstandard2.0;net8.0</TargetFrameworks>
		<Nullable>enable</Nullable>
		<ImplicitUsings>enable</ImplicitUsings>
	</PropertyGroup>
</Project>
```

Lors de la compilation (`dotnet build`), le SDK produit alors :

- un assembly pour `netstandard2.0` (compatible .NET Framework 4.6.1+, .NET Core 2+, .NET 5+),
- un assembly pour `net8.0` (accès aux API modernes de .NET 8).

# Quand multi‑cibler ?

Le multi‑ciblage est particulièrement utile pour :

- des bibliothèques NuGet **grand public** qui doivent rester compatibles avec des projets legacy (.NET Framework) tout en tirant parti des nouveautés de .NET moderne ;
- des composants internes partagés entre plusieurs applications ayant des niveaux de mise à jour différents ;
- une migration progressive : conserver un TFM plus ancien (ex. `netstandard2.0`) le temps que toutes les applications clientes soient migrées.

Dans le cas d’une application (exécutable), le multi‑ciblage est plus rare. Il est surtout utilisé pour :

- produire plusieurs variantes de l’application (par exemple, une version console `net8.0` et une version tool `net6.0`),
- fournir un même outil compatible avec plusieurs runtimes (scénarios plus avancés).

# TFMs courants pour le multi‑ciblage

Quelques TFMs fréquemment combinés :

- `netstandard2.0` : socle large, compatible avec .NET Framework 4.6.1+, .NET Core 2+, .NET 5+ ;
- `netstandard2.1` : plus riche, mais non compatible avec .NET Framework ;
- `net6.0`, `net7.0`, `net8.0` : versions du .NET moderne (à privilégier pour les nouveaux projets) ;
- `net48` : .NET Framework 4.8 (si une compatibilité forte avec du legacy est nécessaire).

Exemples de combinaisons typiques :

- Bibliothèque très large compatibilité :

	```xml
	<TargetFrameworks>netstandard2.0;net8.0</TargetFrameworks>
	```

- Bibliothèque ciblant legacy + moderne sans .NET Standard :

	```xml
	<TargetFrameworks>net48;net8.0</TargetFrameworks>
	```

# Compiler des références différentes selon le framework

Certaines dépendances n’existent que pour un TFM donné (par exemple, une librairie ne supporte que `net8.0`). Il est alors possible d’avoir des **références conditionnelles**.

Exemple :

```xml
<Project Sdk="Microsoft.NET.Sdk">
	<PropertyGroup>
		<TargetFrameworks>netstandard2.0;net8.0</TargetFrameworks>
	</PropertyGroup>

	<!-- Référence commune à tous les frameworks -->
	<ItemGroup>
		<PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
	</ItemGroup>

	<!-- Références spécifiques à net8.0 uniquement -->
	<ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
		<PackageReference Include="Some.Net8Only.Package" Version="1.0.0" />
	</ItemGroup>
</Project>
```

Le projet compilera alors :

- pour `netstandard2.0` : uniquement avec `Newtonsoft.Json` ;
- pour `net8.0` : avec `Newtonsoft.Json` + `Some.Net8Only.Package`.

# Code spécifique par framework : compilation conditionnelle

Il arrive qu’un petit morceau de code doive être différent selon le framework cible (API absente en `netstandard2.0`, optimisation spécifique en `net8.0`, etc.). Dans ce cas, on peut utiliser les **directives de compilation conditionnelle**.

Les projets multi‑ciblés définissent automatiquement des symboles comme :

- `NETSTANDARD2_0`,
- `NET6_0`,
- `NET8_0`,
- etc.

Exemple dans le code C# :

```csharp
public static string GetRuntimeDescription()
{
#if NET8_0
		return $".NET 8 - {System.Runtime.InteropServices.RuntimeInformation.FrameworkDescription}";
#elif NETSTANDARD2_0
		return ".NET Standard 2.0 compatible runtime";
#else
		return "Autre runtime .NET";
#endif
}
```

Ainsi, chaque build pour un TFM donné embarque uniquement le code qui le concerne.

# Bonnes pratiques pour un projet multi‑ciblé

- **Limiter les différences de code** entre frameworks : idéalement, la majeure partie du code reste commune. Le code conditionnel doit rester l’exception.  
- **Commencer par le TFM le plus ancien** (ex. `netstandard2.0`) pour définir le “plus petit dénominateur commun” des API utilisées, puis ajouter des optimisations pour les TFMs plus récents.  
- **Documenter les TFMs supportés** dans le README et le fichier `.csproj` (utile pour les consommatrices et consommateurs de la bibliothèque).  
- **Surveiller les références NuGet** : vérifier que les packages sont compatibles avec tous les TFMs ciblés, ou les isoler dans des `ItemGroup` conditionnels.

Le multi‑ciblage permet ainsi de maintenir une seule base de code tout en produisant plusieurs variantes de la bibliothèque, adaptées à différents runtimes .NET.

# Générer un package NuGet multi‑target

Une fois le multi‑ciblage configuré dans le projet, la génération d’un package NuGet va automatiquement inclure **un assembly par framework cible** dans le même `.nupkg`.

## Activer la génération de package

Pour une bibliothèque multi‑ciblée, la configuration minimale dans le `.csproj` ressemble à ceci :

```xml
<Project Sdk="Microsoft.NET.Sdk">
	<PropertyGroup>
		<TargetFrameworks>netstandard2.0;net8.0</TargetFrameworks>

		<!-- Métadonnées NuGet principales -->
		<GeneratePackageOnBuild>true</GeneratePackageOnBuild>
		<PackageId>MonEntreprise.MaLibrairie</PackageId>
		<Version>1.0.0</Version>
		<Authors>MonEntreprise</Authors>
		<Company>MonEntreprise</Company>
		<Description>Bibliothèque multi‑target compatible .NET Standard 2.0 et .NET 8.</Description>
		<RepositoryUrl>https://github.com/monentreprise/monrepo</RepositoryUrl>
		<PackageProjectUrl>https://monentreprise.fr/ma-librairie</PackageProjectUrl>
	</PropertyGroup>
</Project>
```

Avec `GeneratePackageOnBuild` à `true`, un `dotnet build` dans le dossier du projet génère déjà un `.nupkg` (dans `bin/<Configuration>`). Il est également possible d’utiliser explicitement :

- `dotnet pack` (configuration par défaut),
- ou `dotnet pack -c Release` pour produire un package en mode Release.

Le package contiendra alors, par exemple :

- `lib/netstandard2.0/MonEntreprise.MaLibrairie.dll`,
- `lib/net8.0/MonEntreprise.MaLibrairie.dll`.

Les projets consommateurs choisiront automatiquement l’assembly correspondant à leur TFM.

## Dépendances et multi‑target dans le package

Les références NuGet conditionnelles vues plus haut sont également prises en compte lors du `pack` :

- les dépendances communes sont listées une seule fois,
- les dépendances spécifiques à un TFM apparaissent uniquement pour ce TFM dans le manifeste du package.

Il est donc recommandé de :

- vérifier le contenu du `.nupkg` avec un outil comme `nuget.exe` (`nuget.exe list -Source ...`) ou un explorateur de packages,  
- tester l’installation du package dans plusieurs types de projets (par exemple, un projet `net48` et un projet `net8.0`) pour s’assurer que la bonne variante est bien sélectionnée.

Une fois le package validé en local, la publication se fait de façon classique avec :

```bash
dotnet nuget push MonEntreprise.MaLibrairie.1.0.0.nupkg \
	--api-key <CLE_API> \
	--source https://api.nuget.org/v3/index.json
```

La multi‑cible étant déjà décrite dans le `.csproj`, aucun paramètre spécifique supplémentaire n’est nécessaire pour que NuGet comprenne qu’il s’agit d’un package multi‑framework.

