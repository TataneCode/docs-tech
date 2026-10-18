# Shell Linux — cheat sheet développeur

> Catégorie : Cheat sheet

**Date :** 2026-10-18
**Statut :** Brouillon

## Objectif

Retrouver en quelques secondes les commandes shell du quotidien de développeur : permissions et exécution, recherche dans les logs, redirections de sortie — orienté recherche rapide, pas tutoriel.

## Prérequis

- Un terminal Linux (base Debian/Ubuntu testée)
- Aucun prérequis de niveau

## Navigation et fichiers

| Commande | Rôle | Exemple |
|----------|------|---------|
| `ls -lah` | Lister avec fichiers cachés, tailles lisibles | `ls -lah /var/log` |
| `cd -` | Retourner au dossier précédent | `cd -` |
| `pwd` | Afficher le dossier courant | `pwd` |
| `find` | Chercher des fichiers | `find . -name "*.log"` |
| `find` (taille) | Fichiers > 100 Mo | `find . -size +100M` |
| `tree -L 2` | Arborescence sur 2 niveaux | `tree -L 2` |
| `mkdir -p` | Créer une arborescence complète | `mkdir -p a/b/c` |

## Permissions et exécution

| Commande | Rôle | Exemple |
|----------|------|---------|
| `chmod +x` | Rendre un fichier exécutable | `chmod +x script.sh` |
| `chmod` (octal) | Fixer des droits précis | `chmod 755 script.sh` |
| `chmod -R` | Récursif sur un dossier | `chmod -R 755 bin/` |
| `./script.sh` | Exécuter le script du dossier courant | `./script.sh` |
| `chown` | Changer le propriétaire | `sudo chown user:groupe file` |
| `ls -l` | Voir les droits | `ls -l script.sh` |

Notation octale : lecture=4, écriture=2, exécution=1. `755` = propriétaire tout, groupe et autres lecture+exécution. `644` = standard pour un fichier non exécutable.

## Recherche dans les fichiers (grep)

| Commande | Rôle | Exemple |
|----------|------|---------|
| `grep motif` | Chercher dans un fichier | `grep ERROR app.log` |
| `grep -n` | Avec numéros de ligne | `grep -n ERROR app.log` |
| `grep -i` | Ignorer la casse | `grep -i error app.log` |
| `grep -r` | Récursif dans un dossier | `grep -rn "TODO" src/` |
| `grep -v` | Inverser (exclure le motif) | `grep -v DEBUG app.log` |
| `grep -E` | Regex étendue | `grep -E "ERROR|WARN" app.log` |
| `grep -c` | Compter les occurrences | `grep -c ERROR app.log` |
| `tail -f` | Suivre un log en direct | `tail -f app.log` |
| `tail -n 100` | 100 dernières lignes | `tail -n 100 app.log` |
| `head -n 20` | 20 premières lignes | `head -n 20 app.log` |
| `wc -l` | Compter les lignes | `grep ERROR app.log \| wc -l` |
| `find + grep` | Grep sur des fichiers trouvés | `find . -name "*.log" -exec grep -l ERROR {} +` |

## Redirections et pipes

| Syntaxe | Rôle | Exemple |
|---------|------|---------|
| `>` | Écraser la sortie dans un fichier | `./app > out.txt` |
| `>>` | Ajouter à la fin d'un fichier | `./app >> out.txt` |
| `2>` | Rediriger les erreurs | `./app 2> err.txt` |
| `2>&1` | Erreurs vers la même sortie | `./app > out.txt 2>&1` |
| `2>/dev/null` | Ignorer les erreurs | `./app 2>/dev/null` |
| `tee` | Afficher ET écrire dans un fichier | `./app \| tee out.txt` |
| `tee -a` | Idem, en ajout | `./app \| tee -a out.txt` |
| `<` | Lire l'entrée depuis un fichier | `./app < input.txt` |
| `\|` | Chaîner les commandes | `ps aux \| grep nginx` |

## Processus et disque

| Commande | Rôle | Exemple |
|----------|------|---------|
| `ps aux` | Lister les processus | `ps aux \| grep python` |
| `pgrep` | PID d'un processus par nom | `pgrep -f "python app.py"` |
| `kill` | Terminer un processus | `kill 1234` |
| `kill -9` | Forcer (dernier recours) | `kill -9 1234` |
| `pkill` | Tuer par nom | `pkill -f "python app.py"` |
| `df -h` | Espace disque par partition | `df -h` |
| `du -sh` | Taille d'un dossier | `du -sh ./logs` |
| `htop` | Moniteur interactif | `htop` |

## One-liners utiles

```bash
# Nombre d'erreurs dans un log
grep -c ERROR app.log

# Erreurs avec contexte : 2 lignes avant/après
grep -B2 -A2 ERROR app.log

# Trier les erreurs par fréquence (top motifs)
grep ERROR app.log | sort | uniq -c | sort -rn | head

# PID d'un port occupé
lsof -i :8080

# Vider un fichier sans le supprimer (utile pour un log ouvert)
> app.log

# Exécuter et journaliser tout (sortie + erreurs) horodaté
./app 2>&1 | tee "$(date +%F-%T).log"
```

## Pièges connus

- `>` écrase sans confirmation — préférer `>>` en cas de doute.
- `./app 2>&1 > out.txt` ne fait pas ce qu'on croit : l'ordre compte, écrire `./app > out.txt 2>&1`.
- `grep motif dossier/` sans `-r` cherche les fichiers du dossier en texte, pas à l'intérieur ; utiliser `grep -r`.
- `kill -9` en premier réflexe empêche le process de nettoyer derrière lui ; tenter `kill` simple d'abord.
- Un script `.sh` non exécutable : `bash script.sh` marche, mais vérifier le `chmod +x` et le shebang (`#!/bin/bash`) pour l'exécution directe.

## Références

- [man bash](https://www.gnu.org/software/bash/manual/bash.html)
- [GNU grep — documentation](https://www.gnu.org/software/grep/manual/grep.html)
- [Coreutils (ls, cp, tee, sort, wc...)](https://www.gnu.org/software/coreutils/manual/)
