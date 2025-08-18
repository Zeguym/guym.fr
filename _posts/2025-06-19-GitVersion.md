---
title: Git Version
tags: dotnet dotnettools nuget 
---

**GitVersion** est un outil open source qui automatise la gestion des versions sémantique (SemVer) dans les projets utilisant Git.

# À quoi sert GitVersion ?

- Il génère automatiquement un numéro de version basé sur l’historique Git, la branche courante et la stratégie de workflow (par exemple GitFlow ou GitHub Flow).
- Il évite de devoir incrémenter manuellement la version dans vos fichiers.
- Il s’intègre facilement dans les pipelines CI/CD (Azure DevOps, GitHub Actions, etc.) pour versionner vos builds, packages NuGet, ou artefacts.
<!--more-->

# Comment ça marche ?

##Analyse du dépôt Git
GitVersion lit les branches, tags et commits pour déterminer la version à appliquer selon la configuration (par défaut : GitFlow).

**Stratégie de versionnement**

Par défaut GitVersion suit la stratégie de Gitflow

- Sur `main` : version stable (ex : 1.2.0)
- Sur `release/1.3.0` : version release (ex: 1.3.0-beta.1)
- Sur `develop` : version pré-release (ex : 1.3.0-alpha.5)
- Sur une branche `feature/xyz` : version snapshot (ex : 1.3.0-feature-xyz.1)

## Sortie

GitVersion peut générer :

- Des variables d’environnement (pour CI/CD)
- Un fichier `version.json` ou `AssemblyInfo.cs`
- Des propriétés MSBuild pour .NET

## Exemple d’utilisation

- **Installation** :

installation locale :

```dotnetcli
dotnet tool install --global GitVersion.Tool
```

Installation au niveau de la solution dotnet

```dotnetcli
dotnet new tool-manifest
dotnet tool install GitVersion.Tool
```

- **Afficher la version de la branche en cours** :

```dotnetcli
dotnet gitversion
```

- **Intégration dans un pipeline** :
  Utilisez la version générée pour nommer vos packages, images Docker, etc.

## Configuration

Ajoutez un fichier `GitVersion.yml` à la racine du repo pour personnaliser le comportement (stratégie GitFlow, format des versions, etc.).

Pour plus de details [la doc officielle](https://gitversion.net/docs/reference/configuration)

# GitVersion.MsBuild

`GitVersion.MsBuild` est un package NuGet qui permet d’intégrer automatiquement GitVersion dans le processus de build MSBuild/.NET, sans avoir besoin d’appeler la CLI séparément. Il est particulièrement utile pour les projets .NET qui utilisent `dotnet build`, ou des pipelines CI/CD.

## Fonctionnement

- **Ajout du package**  
  Ajoutez le package NuGet à votre projet ou solution :

```dotnetcli
dotnet add package GitVersion.MsBuild
```

- **Détection automatique**  
  Lors de chaque build (`dotnet build`, `msbuild`, ou build Visual Studio), GitVersion analyse votre dépôt Git et génère la version appropriée selon la configuration (par défaut : GitFlow, personnalisable via `GitVersion.yml`).
- **Injection dans le build**  
  Les propriétés de version générées (par exemple : `Version`, `AssemblyVersion`, `FileVersion`, `PackageVersion`) sont injectées automatiquement dans le build .NET, vos assemblies et vos packages NuGet.

## Avantages

- **Aucune commande supplémentaire à exécuter** : la version est calculée à chaque build.
- **Intégration transparente** : fonctionne avec Visual Studio, `dotnet build`, et les pipelines CI/CD.
- **Versionnement cohérent** : la version de vos DLL, EXE, et packages NuGet reflète toujours l’état du dépôt Git.

> [!WARNING]
> la version 5 de `GitVersion.MsBuild` ne supporte plus dotnet framework, mais uniquement dotnet core. Visual studio 2022 étant en dotnet framework, lors d'une compilation de depuis visual studio il ne se passera rien. Les build depuis la CI via msbuild seront corrects.
