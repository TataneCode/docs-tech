# Vim — cheat sheet développeur

> Catégorie : Cheat sheet

**Date :** 2026-10-18
**Statut :** Brouillon

## Objectif

Consulter et éditer rapidement des fichiers et du code avec Vim : ouvrir les fichiers créés par redirections shell, chercher une erreur dans un log, éditer du code — sans lire un tutoriel complet.

## Prérequis

- Vim installé : `sudo apt install vim`
- Connaître les trois modes (ci-dessous) suffit pour toute la feuille

## Les modes : la clé de Vim

| Mode | Rôle | Comment y entrer | Comment en sortir |
|------|------|------------------|-------------------|
| Normal | Navigation et commandes | `Échap` (touche refuge) | — |
| Insertion | Taper du texte | `i`, `a`, `o` | `Échap` |
| Commande | Sauvegarder, quitter, chercher | `:` depuis Normal | `Échap` |

Réflexe d'or : **en cas de doute, taper `Échap`** pour revenir en mode Normal.

## Survie absolue

| Touche | Action |
|--------|--------|
| `:w` | Sauvegarder |
| `:q` | Quitter |
| `:wq` | Sauvegarder et quitter |
| `:q!` | Quitter sans sauvegarder (abandonner les modifs) |
| `:w nom.txt` | Sauvegarder sous un autre nom |
| `Échap` | Retour au mode Normal en toute circonstance |

## Ouvrir des fichiers

| Commande | Rôle |
|----------|------|
| `vim app.log` | Ouvrir (ou créer) un fichier |
| `vim -R app.log` | Ouvrir en lecture seule |
| `vim -d a.txt b.txt` | Comparer deux fichiers (diff) |
| `vim src/ script.sh` | Ouvrir plusieurs fichiers (`:n` suivant, `:N` précédent) |
| `view app.log` | Alias lecture seule |

## Navigation

| Touche | Action |
|--------|--------|
| `h j k l` | Gauche, bas, haut, droite |
| `w` / `b` | Mot suivant / précédent |
| `0` / `$` | Début / fin de ligne |
| `gg` / `G` | Début / fin du fichier |
| `42G` ou `:42` | Aller à la ligne 42 |
| `{` / `}` | Paragraphe précédent / suivant |
| `Ctrl-d` / `Ctrl-u` | Descendre / montrer d'un demi-écran |
| `Ctrl-e` / `Ctrl-y` | Scroller d'une ligne |
| `zz` | Centrer l'écran sur le curseur |

## Édition

| Touche | Action |
|--------|--------|
| `i` / `a` | Insérer avant/après le curseur |
| `o` / `O` | Nouvelle ligne en dessous/dessus |
| `x` | Supprimer un caractère |
| `dd` | Couper la ligne |
| `yy` | Copier la ligne |
| `p` / `P` | Coller après/avant |
| `u` | Annuler |
| `Ctrl-r` | Rétablir |
| `r` | Remplacer un caractère |
| `cw` | Changer le mot sous le curseur |
| `d$` | Couper jusqu'à la fin de ligne |
| `.` | Répéter la dernière action |
| `3dd` | Couper 3 lignes (compteur applicable partout) |

## Recherche et substitution

| Commande | Rôle |
|----------|------|
| `/motif` | Chercher en avant |
| `?motif` | Chercher en arrière |
| `n` / `N` | Occurrence suivante / précédente |
| `*` | Chercher le mot sous le curseur |
| `:%s/ancien/nouveau/g` | Remplacer dans tout le fichier |
| `:%s/ancien/nouveau/gc` | Idem avec confirmation à chaque fois |
| `:s/ancien/nouveau/` | Remplacer sur la ligne courante |

## Pour le développeur

| Commande | Rôle |
|----------|------|
| `:set number` / `:set nu` | Numéros de ligne |
| `:set relativenumber` | Numérotation relative (pratique pour `5j`, `3dd`) |
| `>>` / `<<` | Indenter / désindenter la ligne |
| `=%` | Réindenter le bloc sous le curseur |
| `:sp` / `:vsp` | Split horizontal / vertical |
| `Ctrl-w w` | Basculer entre splits |
| `:e fichier` | Ouvrir un autre fichier |
| `Ctrl-o` / `Ctrl-i` | Position précédente / suivante dans l'historique |
| `:w !sudo tee %` | Sauvegarder un fichier ouvert sans sudo (le piège classique) |

## Config minimale — `~/.vimrc`

```vim
set number
set relativenumber
syntax on
set tabstop=4
set shiftwidth=4
set expandtab
set incsearch
set hlsearch
```

À copier tel quel pour une expérience correcte en édition de code.

## Séquences types

```text
Ouvrir un log généré par redirection et chercher une erreur :
  vim app.log  →  /ERROR  →  n (occurrences suivantes)  →  :q

Éditer un script :
  vim script.sh  →  /motif  →  cw + nouveau texte  →  Échap  →  :wq

Reformater tous les "foo" en "bar" dans un fichier :
  vim file.txt  →  :%s/foo/bar/g  →  :wq
```

## Pièges connus

- **Impossible de quitter** : `Échap` puis `:q!` (abandonner) ou `:wq` (sauvegarder). C'est le piège n°1.
- **Taper du texte en mode Normal** produit des comportements étranges (suppressions, sauts) : taper `Échap` puis `u` pour annuler, puis `i` pour insérer proprement.
- `:q` refuse de quitter si des modifications non sauvegardées existent : c'est voulu, utiliser `:wq` ou `:q!`.
- Coller du texte avec `Ctrl-V` ne fonctionne pas dans Vim : en mode Insertion, le collage terminal (`Ctrl-Shift-V`) insère parfois avec indentation cassée — activer `:set paste` avant de coller, et `:set nopaste` après.
- Sur une machine minimale (conteneur), `vim` peut être absent : `vi` y est souvent en version minimale ; les commandes de base ci-dessus fonctionnent pareil.

## Références

- [Vim — documentation officielle](https://www.vim.org/docs.php)
- `vimtutor` — tutoriel interactif de 30 min, installé avec Vim
- [Learn Vimscript the Hard Way](https://learnvimscriptthehardway.stevelosh.com/)
