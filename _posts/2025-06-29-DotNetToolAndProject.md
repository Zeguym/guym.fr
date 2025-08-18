---
title: Dotnet tools installer des outils par projets
tags: dotnet dotnettools 
---

Comment installer des outils dotnet localement a un projet spécifique et non globalement et partager la configuration avec le projet pour une meilleur reproductibilité

<!--more-->

# Installer des outils spécifique a un projet dotnet

1. **Créer le manifeste d’outils**  
   Dans le dossier de votre projet, exécutez :

   ```
   dotnet new tool-manifest
   ```

   Cela crée un fichier `dotnet-tools.json` qui va lister les outils utilisés localement.

2. **Installer un outil local**  
   Par exemple, pour installer l’outil `dotnet-ef` :

   ```
   dotnet tool install dotnet-ef
   ```

   L’outil est alors disponible uniquement pour ce projet (pas globalement).

3. **Utiliser l’outil**  
   Lancez l’outil avec :
   ```
   dotnet tool run dotnet-ef --help
   ```

**Avantage :**  
Les outils sont versionnés et partagés avec le projet, ce qui facilite la collaboration et la reproductibilité.

# Sytaxe du fichier 'dotnet-tools.json'

Le fichier `dotnet-tools.json` est un manifeste JSON qui liste les outils .NET installés localement pour un projet. Sa structure est la suivante :

```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "dotnet-ef": {
      "version": "8.0.0",
      "commands": ["dotnet-ef"]
    },
    "autre-outil": {
      "version": "x.y.z",
      "commands": ["nom-commande"]
    }
  }
}
```

- **version** : Version du format du manifeste (toujours 1).
- **isRoot** : Indique que ce manifeste est à la racine du projet.
- **tools** : Objet listant chaque outil installé.
- **dotnet-ef** : Nom de l’outil.
- **version** : Version installée de l’outil.
- **commands** : Liste des commandes exposées par l’outil.

Chaque outil ajouté avec `dotnet tool install` apparaît dans la section `tools` avec sa version et ses commandes.

# Restaurer des outils

## Manuellement

Pour restaurer les outils listés dans le fichier `dotnet-tools.json`, utilisez la commande suivante dans le dossier du projet :

```
dotnet tool restore
```

Cette commande installe toutes les versions des outils spécifiées dans le manifeste, assurant que votre environnement local correspond à celui défini pour le projet.

## Automatiquement

Voici comment faire automatiquement un restore a change changement de branche Git. En ce plaçant ans le répertoire racine du projet, ou lon a préalablement fait le `dotnet new tool-manifest`

1. Installer Husky.net :

```
dotnet tool install Husky
```

2. Configurer Husky.net :

```
dotnet husky install
```

3. Ajouter un hook Husky.net pour executer le restore :

```
dotnet husky add post-checkout -c "dotnet tool restore"
```

4. Attachez Husky.net a l'un des projets present dans le dossier. L'ouverture du projet dans visual studio provoquera l'installation de la tool box.

```
dotnet husky attach <path-to-project-file.csproj>
```
