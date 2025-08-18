---
title: "Bonnes pratiques avec les fichiers en .NET"
tags: dotnet fichiers io bonnes-pratiques
---

La manipulation de fichiers est une opération courante qui cache de nombreux pièges : chemins non portables, ressources non libérées, fichiers temporaires oubliés, encodage incorrect… Cet article passe en revue les bonnes pratiques à adopter en .NET.

<!--more-->

## 1. Concaténation de chemins

### Ce qu'il ne faut jamais faire

```csharp
// Fragile : séparateur en dur, non portable Linux/Windows
var path = baseDir + "\\" + "subfolder" + "\\" + "file.txt";

// Fragile : les slashes peuvent ne pas correspondre à l'OS
var path = baseDir + "/subfolder/file.txt";
```

### `Path.Combine` — la référence

`Path.Combine` gère automatiquement les séparateurs selon l'OS :

```csharp
var path = Path.Combine(baseDir, "subfolder", "file.txt");
```

Attention : si un des segments est un chemin absolu, `Path.Combine` **abandonne** les segments précédents.

```csharp
Path.Combine("C:\\data", "C:\\other", "file.txt"); // → "C:\\other\\file.txt"
```

### `Path.Join` (préféré depuis .NET 5)

`Path.Join` ne souffre pas de ce comportement : il concatène toujours tous les segments et ne tronque jamais :

```csharp
var path = Path.Join(baseDir, "subfolder", "file.txt");

// Avec des Span<char> pour éviter les allocations
var path = Path.Join(baseDir.AsSpan(), "subfolder".AsSpan(), "file.txt".AsSpan());
```

> **Règle** : Utilisez `Path.Join` pour construire des chemins, `Path.Combine` uniquement si vous voulez explicitement qu'un segment absolu écrase les précédents.

### Normaliser un chemin

```csharp
// Résout les ".." et les doubles séparateurs
var normalized = Path.GetFullPath(path);

// Résoudre relativement à une base
var full = Path.GetFullPath("../config.json", baseDir);
```

---

## 2. Lire et écrire des fichiers

### `File` — méthodes statiques pour les petits fichiers

```csharp
// Lecture complète
string content = await File.ReadAllTextAsync(path, Encoding.UTF8);
byte[] bytes    = await File.ReadAllBytesAsync(path);
string[] lines  = await File.ReadAllLinesAsync(path, Encoding.UTF8);

// Écriture complète
await File.WriteAllTextAsync(path, content, Encoding.UTF8);
await File.WriteAllBytesAsync(path, bytes);
await File.WriteAllLinesAsync(path, lines, Encoding.UTF8);
```

### `StreamReader` / `StreamWriter` pour les gros fichiers

Pour les fichiers volumineux, lisez ligne par ligne pour éviter de tout charger en mémoire :

```csharp
await using var reader = new StreamReader(path, Encoding.UTF8);
while (!reader.EndOfStream)
{
    var line = await reader.ReadLineAsync();
    // traiter line
}
```

Écriture bufferisée :

```csharp
await using var writer = new StreamWriter(path, append: false, Encoding.UTF8);
await writer.WriteLineAsync("première ligne");
await writer.WriteLineAsync("deuxième ligne");
// flush automatique à la fin du using
```

### `FileStream` pour un contrôle fin

```csharp
await using var fs = new FileStream(
    path,
    FileMode.Create,
    FileAccess.Write,
    FileShare.None,
    bufferSize: 4096,
    useAsync: true);

await fs.WriteAsync(buffer.AsMemory(0, bytesRead));
```

---

## 3. Toujours libérer les ressources

Toute classe implémentant `IDisposable` ou `IAsyncDisposable` doit être utilisée dans un bloc `using` :

```csharp
// Synchrone
using var stream = File.OpenRead(path);

// Asynchrone — préférez await using
await using var stream = File.OpenReadAsync(path);
```

Ne jamais faire :

```csharp
var stream = File.OpenRead(path);
// ... oubli de Dispose → le fichier reste verrouillé
```

---

## 4. Fichiers temporaires

### `Path.GetTempFileName` — le piège courant

```csharp
// Crée physiquement un fichier de 0 octet dans le dossier Temp de l'OS
// et retourne son chemin — risque de collision si mal géré
var tempFile = Path.GetTempFileName();
```

Le problème : si votre programme plante, le fichier reste sur disque.

### Pattern recommandé

Créez vos propres fichiers temporaires avec un nom unique et nettoyez-les explicitement :

```csharp
var tempPath = Path.Combine(Path.GetTempPath(), $"{Guid.NewGuid()}.tmp");
try
{
    await File.WriteAllTextAsync(tempPath, content);
    // ... traitements
}
finally
{
    File.Delete(tempPath); // garanti même en cas d'exception
}
```

### Encapsuler dans un `IDisposable`

Pour une gestion propre et réutilisable :

```csharp
public sealed class TempFile : IDisposable
{
    public string Path { get; } = System.IO.Path.Combine(
        System.IO.Path.GetTempPath(),
        $"{Guid.NewGuid()}.tmp");

    public void Dispose()
    {
        if (File.Exists(Path))
            File.Delete(Path);
    }
}

// Utilisation
using var temp = new TempFile();
await File.WriteAllTextAsync(temp.Path, data);
// Suppression automatique à la sortie du using
```

---

## 5. Encodage

Spécifiez **toujours** l'encodage explicitement. Ne jamais se fier à l'encodage par défaut de la machine :

```csharp
// BOM inclus par défaut avec new UTF8Encoding(true)
await File.WriteAllTextAsync(path, content, Encoding.UTF8);

// Sans BOM (courant pour les échanges inter-systèmes)
await File.WriteAllTextAsync(path, content, new UTF8Encoding(encoderShouldEmitUTF8Identifier: false));
```

Pour détecter l'encodage d'un fichier inconnu, utilisez un package comme `Ude.NetStandard` ou `UTF8Encoding.Default` en dernier recours.

---

## 6. Vérifications d'existence

```csharp
// Fichier
if (File.Exists(path))
{
    // ...
}

// Répertoire
if (Directory.Exists(dirPath))
{
    // ...
}

// Créer le répertoire si absent (idempotent)
Directory.CreateDirectory(dirPath);
```

> `Directory.CreateDirectory` ne lève pas d'exception si le dossier existe déjà. Pas besoin de vérifier au préalable.

---

## 7. Créer le chemin complet d'un fichier s'il n'existe pas

Avant d'écrire dans un fichier, son **répertoire parent** doit exister. `File.WriteAllTextAsync` ne crée pas les dossiers intermédiaires et lèvera une `DirectoryNotFoundException` si l'un d'eux manque.

Le pattern à adopter :

```csharp
var filePath = Path.Join(baseDir, "reports", "2026", "rapport.pdf");

// Crée tous les répertoires intermédiaires en une seule instruction (idempotent)
Directory.CreateDirectory(Path.GetDirectoryName(filePath)!);

await File.WriteAllBytesAsync(filePath, pdfBytes);
```

`Path.GetDirectoryName` extrait la partie répertoire du chemin complet, et `Directory.CreateDirectory` crée récursivement tous les segments manquants.

Encapsuler cette logique dans une méthode utilitaire évite de l'oublier :

```csharp
public static async Task WriteAllTextSafeAsync(string filePath, string content, Encoding? encoding = null)
{
    Directory.CreateDirectory(Path.GetDirectoryName(filePath)!);
    await File.WriteAllTextAsync(filePath, content, encoding ?? Encoding.UTF8);
}
```

> Attention au `!` sur `Path.GetDirectoryName` : la méthode retourne `null` uniquement si le chemin est une racine (`"C:\\"` ou `"/"`). Dans tous les autres cas le résultat est non nul. Si votre chemin peut être une racine, ajoutez une vérification explicite.

---

## 8. Sécurité des chemins (Path Traversal)

Si le chemin est fourni par un utilisateur, **validez-le** pour éviter les attaques de type path traversal :

```csharp
public string GetSecurePath(string baseDir, string userInput)
{
    // Normaliser pour résoudre les ".."
    var fullPath = Path.GetFullPath(Path.Join(baseDir, userInput));

    // Vérifier que le chemin résultant reste sous baseDir
    if (!fullPath.StartsWith(Path.GetFullPath(baseDir) + Path.DirectorySeparatorChar,
                              StringComparison.OrdinalIgnoreCase))
    {
        throw new UnauthorizedAccessException("Accès refusé : chemin hors du répertoire autorisé.");
    }

    return fullPath;
}
```

---

## 9. Surveillance de fichiers avec `FileSystemWatcher`

```csharp
using var watcher = new FileSystemWatcher(dirPath)
{
    Filter = "*.json",
    NotifyFilter = NotifyFilters.LastWrite | NotifyFilters.FileName,
    IncludeSubdirectories = false,
    EnableRaisingEvents = true
};

watcher.Changed += (_, e) => Console.WriteLine($"Modifié : {e.FullPath}");
watcher.Created += (_, e) => Console.WriteLine($"Créé    : {e.FullPath}");
watcher.Deleted += (_, e) => Console.WriteLine($"Supprimé: {e.FullPath}");

// Éviter les événements en double (le Changed se déclenche parfois deux fois)
watcher.Changed += OnChangedDebounced;
```

> `FileSystemWatcher` peut déclencher plusieurs événements pour une seule modification (selon l'OS). Implémentez un **debounce** si nécessaire.

---

## 10. Opérations sur les répertoires

```csharp
// Lister les fichiers (non récursif)
var files = Directory.GetFiles(dirPath, "*.txt");

// Lister récursivement sans tout charger en mémoire
foreach (var file in Directory.EnumerateFiles(dirPath, "*.txt", SearchOption.AllDirectories))
{
    // traiter file
}

// Copier un répertoire entier (non intégré nativement avant .NET 8)
// À partir de .NET 8 :
// Pas d'API built-in, mais on peut utiliser DirectoryInfo
new DirectoryInfo(sourceDir).CopyTo(destinationDir); // ⚠ n'existe pas

// Pattern manuel
foreach (var file in Directory.EnumerateFiles(sourceDir, "*", SearchOption.AllDirectories))
{
    var relativePath = Path.GetRelativePath(sourceDir, file);
    var destFile     = Path.Join(destinationDir, relativePath);
    Directory.CreateDirectory(Path.GetDirectoryName(destFile)!);
    File.Copy(file, destFile, overwrite: true);
}
```

Préférez `Directory.EnumerateFiles` à `Directory.GetFiles` pour les grands répertoires : le premier est en *streaming* et n'alloue pas un tableau complet en mémoire.

---

## 11. Portabilité multi-OS

| À éviter | À préférer |
|----------|-----------|
| `"\\"` ou `"/"` en dur | `Path.DirectorySeparatorChar` ou `Path.Join` |
| Comparaison de chemins avec `==` | `string.Equals(a, b, StringComparison.OrdinalIgnoreCase)` sur Windows, `Ordinal` sur Linux |
| Chemins codés en dur (`C:\\data`) | `AppContext.BaseDirectory`, `Environment.GetFolderPath`, `IWebHostEnvironment.ContentRootPath` |
| `Path.GetTempFileName` sans nettoyage | Pattern `TempFile` avec `IDisposable` |

```csharp
// Emplacement portable pour les données d'application
var appData = Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData);
var configPath = Path.Join(appData, "MonApp", "config.json");
```

---

## Récapitulatif

| Pratique | API / Pattern |
|----------|--------------|
| Construire un chemin | `Path.Join` |
| Normaliser un chemin | `Path.GetFullPath` |
| Petits fichiers | `File.ReadAllTextAsync` / `WriteAllTextAsync` |
| Gros fichiers | `StreamReader` / `StreamWriter` en ligne par ligne |
| Libérer les ressources | `await using` / `using` |
| Fichiers temporaires | `TempFile : IDisposable` + `finally` |
| Encodage | Toujours explicite (`Encoding.UTF8`) |
| Créer le chemin complet | `Directory.CreateDirectory(Path.GetDirectoryName(path)!)` |
| Sécurité chemin | `Path.GetFullPath` + vérification du préfixe |
| Surveillance | `FileSystemWatcher` + debounce |
| Lister un dossier | `Directory.EnumerateFiles` (streaming) |

La gestion des fichiers est un domaine qui semble simple mais concentre de nombreux bugs en production. Quelques habitudes bien ancrées — `Path.Join`, `await using`, nettoyage systématique des temporaires et validation des chemins utilisateurs — suffisent à éviter la grande majorité des problèmes.
