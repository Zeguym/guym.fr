---
title: Nerd terminal
tags: terminal powershell ohmyposh psreadline
---

Améliorez l’apparence et les fonctionnalités de votre terminal PowerShell sous Windows avec une police adaptée, Oh My Posh pour le prompt, et PSReadLine pour un historique beaucoup plus efficace.

<!--more-->

![nerdterminal](/assets/images/nerdterminal.png)

# Police de caractères

Microsoft Cascadia Code sous Windows, notamment dans VS Code et le Terminal.

1. Installer la police

https://github.com/microsoft/cascadia-code/releases

- Variants: Cascadia Code (ligatures), Cascadia Mono (sans ligatures), Cascadia Code PL (glyphes Powerline).

2. VS Code (éditeur)

- Fichier > Préférences > Paramètres > “Font Family” = Cascadia Code
- Activer “Font Ligatures”
- Ou via settings.json:

```json
{
  // ...existing code...
  "editor.fontFamily": "Cascadia Code, Consolas, 'Courier New', monospace",
  "editor.fontLigatures": true
  // ...existing code...
}
```

3. VS Code (terminal intégré)

- Paramètre:

```json
{
  // ...existing code...
  "terminal.integrated.fontFamily": "Cascadia Code PL"
  // ...existing code...
}
```

4. Windows Terminal

- Paramètres > Profiles > Defaults > Appearance > Font face: “Cascadia Code PL” (ou “Cascadia Code”/“Cascadia Mono” selon besoin).

5. Visual Studio

- Tools > Options > Environment > Fonts and Colors > Text Editor > Font: “Cascadia Code NF”.

Note: relancer les applications après l’installation de la police.

# Terminal avec Oh My Posh

1. Installer

- Oh My Posh:

```powershell
winget install JanDeDobbeleer.OhMyPosh -s winget
```

- Police Nerd Font (pour les icônes). Recommandé: CaskaydiaCove (Cascadia patchée):

```powershell
winget install NerdFonts.CaskaydiaCove
```

2. Définir la police dans vos terminaux

- VS Code (terminal intégré) dans settings.json:

```json
{
  // ...existing code...
  "terminal.integrated.fontFamily": "CaskaydiaCove Nerd Font"
  // ...existing code...
}
```

- Windows Terminal: Paramètres > Profiles > Defaults > Appearance > Font face = “CaskaydiaCove Nerd Font”.

3. Initialiser dans PowerShell

- Créer/ouvrir le profil:

```powershell
if (!(Test-Path -Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force }
notepad $PROFILE
```

- Ajouter cette ligne puis enregistrer:

```powershell
oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH\jandedobbeleer.omp.json" | Invoke-Expression
```

4. Tester et changer de thème

- Lister les thèmes:

```powershell
Get-ChildItem $env:POSH_THEMES_PATH
5. Mise à jour

```powershell
winget upgrade JanDeDobbeleer.OhMyPosh
```

# Historique du terminal avec PSReadLine

Après l’apparence du terminal, PSReadLine permet d’améliorer fortement le confort d’utilisation : édition de ligne, autocomplétion, prédictions et recherche d’historique.

1. Installer ou mettre à jour (PowerShell 5+ ou 7+)

```powershell
Install-Module PSReadLine -Scope CurrentUser -Force
```

2. Raccourcis utiles au quotidien

- Tab : menu de complétion.
- Flèche droite : accepter la prédiction.
- Alt+F : accepter le prochain mot suggéré.
- Taper un préfixe puis ↑/↓ : recherche d’historique par préfixe.
- Ctrl+R : recherche incrémentale dans l’historique.

3. Configuration conseillée dans `$PROFILE`

Ouvrir/éditer le profil :

```powershell
if (!(Test-Path -Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force }
notepad $PROFILE
```

Puis ajouter (et redémarrer le terminal) :

```powershell
Import-Module PSReadLine
Set-PSReadLineOption -EditMode Windows
Set-PSReadLineOption -PredictionSource HistoryAndPlugin
Set-PSReadLineOption -PredictionViewStyle ListView
Set-PSReadLineOption -BellStyle None

Set-PSReadLineKeyHandler -Key UpArrow -Function HistorySearchBackward
Set-PSReadLineKeyHandler -Key DownArrow -Function HistorySearchForward
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete
Set-PSReadLineKeyHandler -Key RightArrow -Function AcceptSuggestion
Set-PSReadLineKeyHandler -Key "Alt+f" -Function AcceptNextSuggestionWord
```

4. Optionnel : prédicteur Azure

```powershell
Install-Module Az.Tools.Predictor -Scope CurrentUser
Import-Module Az.Tools.Predictor
Enable-AzPredictor -AllSession
```

5. Diagnostic rapide

```powershell
Get-PSReadLineOption
Get-PSReadLineKeyHandler | Out-Host -Paging
```

En combinant une police adaptée, Oh My Posh pour un prompt riche et PSReadLine pour l’édition et l’historique, votre terminal PowerShell devient un véritable outil de travail confortable et efficace.

- Diagnostic

```powershell
Get-PSReadLineOption
Get-PSReadLineKeyHandler | Out-Host -Paging
```
