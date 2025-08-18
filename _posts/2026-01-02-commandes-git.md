---
layout: article
title: "Commandes Git avancées (et méconnues)"
tags: git
---

Les commandes `git status`, `git commit` ou `git push` sont bien connues. Mais Git cache aussi une série d’outils plus avancés, très utiles au quotidien et pourtant rarement utilisés.

<!--more-->

Voici une sélection commentée, avec des exemples concrets.

## `git worktree` — travailler sur plusieurs branches en parallèle

`git worktree` permet d’avoir plusieurs copies de travail d’un même dépôt, tout en partageant le même dossier `.git`. Idéal pour :

- tester une PR sans fermer votre branche en cours ;
- corriger un hotfix tout en gardant le contexte d’une feature ouverte ;
- éviter les commits « WIP » juste pour pouvoir changer de branche.

**Exemple : créer un worktree pour une nouvelle branche**

```bash
# Depuis le dépôt existant
git worktree add ../feature-x feature/x

# Résultat :
# - le dossier ../feature-x est créé
# - il pointe sur la branche feature/x
```

##  `git rerere` — réutiliser vos résolutions de conflits

`rerere` (REuse REcorded REsolution) enregistre la façon dont vous résolvez un conflit de fusion, et peut la rejouer automatiquement lorsqu’un conflit similaire se représente.

**Activation globale**

```bash
git config --global rerere.enabled true
```

Ensuite, à chaque fois que vous rencontrez deux fois le même conflit (rebase récurrent, branche long‑lived, etc.), Git appliquera automatiquement la résolution déjà enregistrée.

## `git commit --fixup` + `git rebase --autosquash` — corriger proprement l’historique

Au lieu d’empiler des commits « fix typo », « small fix », vous pouvez préparer des commits de correction qui seront automatiquement fusionnés (squash) avec le bon commit lors d’un rebase.

**Exemple de workflow**

```bash
# 1. Vous repérez un bug introduit par un commit existant
git log --oneline
# ... repérez le SHA: 1234abcd

# 2. Vous préparez un commit de correction ciblé
git commit --fixup 1234abcd

# 3. Vous réécrivez l’historique proprement
git rebase -i --autosquash main
```

Après le rebase, votre historique est propre, sans commits de correction bruités.

##  `git range-diff` — comparer deux versions d’une même série de commits

Très utile pour revoir la « v2 » d’une PR après un rebase ou des retouches importantes.

```bash
# Comparer deux séries de commits
git range-diff main..feature-v1 main..feature-v2
```

Git affiche les différences de contenu entre les deux séries, et pas seulement les SHAs.

## `git bisect` — trouver le commit qui a introduit un bug

`git bisect` fait une recherche binaire dans l’historique pour isoler le premier commit fautif.

```bash
git bisect start
git bisect bad              # HEAD est buggé
git bisect good v1.0.0      # cette version fonctionnait

# Git vous place automatiquement au milieu de l’intervalle
# Vous testez, puis :
git bisect good             # ou git bisect bad

# Répétez jusqu’à ce que Git annonce le commit coupable
git bisect reset            # revenir à l’état initial
```

## `git replace` — remplacer temporairement un commit

`git replace` permet d’indiquer à Git qu’un commit est « remplacé » par un autre, localement, sans modifier l’historique du dépôt distant.

Cas d’usage :

- expérimenter un split de dépôt ou un rééchantillonnage d’historique ;
- tester un refactoring profond en gardant des liens avec l’historique existant ;
- assembler virtuellement deux historiques.

```bash
git replace ANCIEN_SHA NOUVEAU_SHA
```

Toutes les commandes d’historique (`log`, `blame`, etc.) verront `NOUVEAU_SHA` à la place de `ANCIEN_SHA`, localement.

## `git notes` — annoter les commits sans changer leur SHA

`git notes` ajoute des métadonnées à un commit (commentaires, liens, références), sans modifier le commit lui‑même.

```bash
# Ajouter une note au commit courant
git notes add -m "Revu par Alice, OK pour prod"

# Afficher les notes dans le log
git log --show-notes
```

Pratique pour stocker des informations de revue, des liens vers un ticket, etc., sans polluer le message de commit.

## `git reflog` — remonter dans tous vos mouvements de HEAD

`git reflog` enregistre chaque mouvement de `HEAD` (checkout, reset, rebase, etc.). C’est souvent la dernière chance pour retrouver un travail « perdu ».

```bash
git reflog

# Une entrée typique
# d2f3e12 HEAD@{3}: rebase finished: returning to refs/heads/feature/x

# Pour retrouver un ancien état
git checkout d2f3e12
# ou recréer une branche
git branch recovery d2f3e12
```

## `git show :/"texte"` — retrouver un commit par son message

Cette syntaxe permet de chercher un commit à partir d’un morceau de message.

```bash
git show :/"fix login bug"
```

Git affiche le premier commit dont le message contient cette chaîne.

## `git grep` — rechercher dans le code versionné

`git grep` est souvent plus rapide et plus précis qu’un simple `grep` sur le système de fichiers, et fonctionne bien même dans de gros dépôts.

```bash
git grep "StackOverflowException"          # recherches simples
git grep -n "DoSomething"                  # avec numéros de lignes
git grep -p "RegexOptions"                 # montre le bloc/fonction autour
```

## `git log` avancé — mieux naviguer dans l’historique

Quelques options très utiles :

```bash
# Vue globale des branches
git log --oneline --graph --decorate --all

# Commits qui ajoutent/suppriment une occurrence donnée
git log -p -S "MySymbol"

# Historique d’un fichier malgré les renommages
git log --follow -- path/vers/fichier.cs
```

## `git add -p` — ajouter les changements hunk par hunk

`git add -p` permet de choisir interactivement quels blocs de modifications (hunks) vous ajoutez au prochain commit.

```bash
git add -p
# Pour chaque bloc : y = ajouter, n = ignorer, s = splitter, e = éditer
```

Idéal pour découper un gros changement en plusieurs commits cohérents.

##  `git stash push -p` — stash sélectif

Vous n’êtes pas obligé de stasher tout le dossier de travail.

```bash
git stash push -p -m "WIP refactoring service"
git stash list
git stash show -p stash@{1}
git stash apply stash@{1}
```

Le `-p` vous laisse choisir hunk par hunk ce qui part dans le stash.

##  `git branch --merged` / `--contains` — nettoyer les branches

```bash
# Branches déjà mergées dans main (candidates à suppression)
git branch --merged main

# Branches qui contiennent un commit donné
git branch --contains <SHA>
```

Tableau des branches deja mergées dans la branche principale avec auteur et date du dernier commit :
```bash
 git --no-pager branch -r --merged master --format='%(HEAD) %(align:80)%(color:yellow)%(refname:short)%(color:reset)%(end) | %(authorname) (%(color:green)%(committerdate:short)%(color:reset))'
```
Pratique pour faire le ménage après plusieurs sprints.

##  `git clean -dn` / `git clean -df` — nettoyer les fichiers non suivis

```bash
git clean -dn   # dry run, montre ce qui serait supprimé
git clean -df   # supprime réellement les fichiers/dossiers non suivis
```

> Attention : cette commande est destructive. Utilisez toujours d’abord `-dn` pour vérifier ce qui va être supprimé.

## Differences entre branches : plusieurs approches
Pour comparer `main` avec une autre branche (disons `feature/x`), tu as plusieurs options utiles :

**1. Diff complet entre deux branches (contenu des fichiers)**  
- Depuis n’importe quelle branche :  
  - `git diff main..feature/x`  
  Montre les différences de contenu entre les deux branches (patch).  
- Juste la liste des fichiers impactés :  
  - `git diff --name-status main..feature/x`  

**2. Voir quels commits sont uniquement dans une branche**  
- Commits présents dans `feature/x` mais pas dans `main` :  
  - `git log main..feature/x --oneline`  
- Commits présents dans `main` mais pas dans `feature/x` :  
  - `git log feature/x..main --oneline`  

**3. Utiliser la syntaxe `...` (trois points) pour comparer à partir de l’ancêtre commun**  
- Diff entre les deux branches en partant de leur ancêtre commun :  
  - `git diff main...feature/x`  
- Commits propres à chaque branche depuis l’ancêtre commun :  
  - `git log main...feature/x --oneline --left-right --graph`  

## resetting un commit tout en gardant les changements dans l’index

Dans le cas d'un merge/rebase complexe, il peut être utile de reset un commit tout en gardant les changements dans l'index pour éviter de devoir tout re-stager manuellement.

La commande suivante replace le HEAD de la branche courante sur la branche develop tout en conservant les modifications dans l'index :

```bash
 git reset --soft develop
```
Le résultat est que les modifications apportées par les commits entre la branche courante et develop sont conservées dans l'index, prêtes à être re-commitées si besoin comme si elles venaient d'être ajoutées avec git squash ou git add. 
