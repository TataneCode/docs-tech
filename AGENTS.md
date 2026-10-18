# Contexte du projet — docs-tech

## Objectif

Dépôt personnel de documentation technique centralisée. Un seul point d'entrée pour tous les sujets techniques qui intéressent le propriétaire : guides d'installation, cheat sheets, notes de configuration et de dépannage.

Exemples de sujets couverts : installation de GLM Studio, cheat sheet DIMM (mémoire), Global Linux, etc.

## Propriétaire et langue

- Propriétaire : TataneCode (dépôt privé, usage personnel).
- **Langue de rédaction : français**, y compris les titres, les commentaires de code et les messages de commit.
- Ton : direct et factuel, sans remplissage. S'adresse à un lecteur technique (le propriétaire lui-même ou un développeur).

## Structure du dépôt

```
README.md                     # Accueil + sommaire central (tableau)
AGENTS.md                     # Ce contexte projet
_templates/article-template.md
articles/<sujet>/README.md    # Un dossier par article
skills/<skill>/SKILL.md        # Skills du projet (voir ci-dessous)
.claude/skills/<skill>/SKILL.md  # Copie pour découverte automatique CLI
```

Règles de structure :
- Un article = un dossier `articles/<nom-du-sujet>/` avec un `README.md`.
- Tout nouvel article **doit** être ajouté au sommaire du `README.md` racine (tableau : article, catégorie, description).
- Le nom de dossier est en minuscules, mots séparés par des tirets (kebab-case).

## Catégories

1. **Guides d'installation** — procédures pas à pas.
2. **Cheat sheets** — aide-mémoire condensé, orienté recherche rapide.
3. **Notes & dépannage** — problème rencontré + solution.

## Workflow

- **Pas de pull requests.** Projet strictement privé, non partagé. Tout est livré par des commits directs sur `main`.
- Commits **propres et atomiques** : un commit = une unité de travail cohérente (un article, une correction, une mise à jour du contexte). Pas de commit mélangeant plusieurs sujets ni de commits « WIP ».
- Avant de committer : vérifier `git status` et le diff pour n'inclure que les fichiers voulus, sans résidus ni secrets.
- Messages de commit en français, impératif présent, une ligne concise.
- Toujours vérifier le sommaire racine après ajout/modification/suppression d'un article.

## Style de rédaction

- Markdown pur, pas d'HTML.
- Commandes dans des blocs de code avec le langage annoté (```bash, ```powershell, etc.).
- Chemins et noms de fichiers en `code inline`.
- Les commandes doivent être testables : indiquer l'OS/le contexte quand il compte.
- Un article commence toujours par l'objectif (ce que le lecteur saura faire), puis les prérequis.
- Terminé un article par « Pièges connus » puis « Références » (liens Markdown descriptifs).

## Skills disponibles

Deux skills normalisés (format Agent Skills : YAML frontmatter `name` + `description` en en-tête de `SKILL.md`) sont disponibles dans `skills/` et dupliqués dans `.claude/skills/` :

1. **planifier-doc** (`skills/planifier-doc/SKILL.md`) — planifier la rédaction d'un article : cadrage, audience, plan détaillé, validation avant rédaction.
2. **rediger-doc** (`skills/rediger-doc/SKILL.md`) — rédiger un article conforme au template et au style de ce dépôt.

L'agent (Vibe Code) applique ces skills en suivant les instructions du `SKILL.md` correspondant. Si un agent CLI découvre automatiquement `.claude/skills/`, il peut les invoquer par leur nom ; sinon, lire le fichier `SKILL.md` et suivre ses instructions.

## Ce qu'il ne faut pas faire

- Ne pas modifier la structure sans mettre à jour ce fichier ni le `README.md`.
- Ne pas commiter de secrets (clés API, tokens, `.env`).
- Ne pas rédiger d'article sans validation du plan par le propriétaire.
