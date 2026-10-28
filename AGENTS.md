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
llm-cli/                      # Boîte à outils CLI LLM (voir ci-dessous)
├── 01-synthese-sources.md    # Synthèse des 6 dépôts étudiés (superpowers, karpathy-skills, i-have-adhd, claude-octopus, OpenMAIC, vibe-wise)
├── 02-formats-cibles.md      # Formats supportés par Gemini CLI et Vibe CLI (vérifiés)
├── 03-plan-preparation.md    # Plan d'installation étape par étape
├── skills/<nom>/             # 20 skills portables au standard Agent Skills (FR)
│   ├── SKILL.md              # Instructions du skill
│   └── scripts/              # Scripts Python optionnels (ex. reset.py pour vibe-wise-reset)
├── agents/<nom>.toml         # 7 personas Vibe (à copier dans ~/.vibe/agents/)
├── prompts/<nom>.md          # Prompts système des personas (à copier dans ~/.vibe/prompts/)
└── gemini/commands/<nom>.toml # 8 commandes slash Gemini CLI (à copier dans ~/.gemini/commands/)
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

- **Merge requests obligatoires.** Toute mise à jour de `main` passe désormais par une MR : branche de travail (`vibe/<sujet>` ou nom explicite), commit(s) propres et atomiques, MR ouverte vers `main`, revue puis merge. Plus de commits directs sur `main`.
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

Deux skills normalisés (format Agent Skills : YAML frontmatter `name` + `description` en en-tête de `SKILL.md`) sont disponibles dans `skills/` :

1. **planifier-doc** (`skills/planifier-doc/SKILL.md`) — planifier la rédaction d'un article : cadrage, audience, plan détaillé, validation avant rédaction.
2. **rediger-doc** (`skills/rediger-doc/SKILL.md`) — rédiger un article conforme au template et au style de ce dépôt.

L'agent (Vibe Code ou CLI Vibe) applique ces skills en lisant le fichier `SKILL.md` correspondant dans `skills/` et en suivant ses instructions.

## Dossier `llm-cli/` — boîte à outils CLI LLM

Collection personnelle de skills, personas et commandes prêts à installer sur **Gemini CLI** et **Vibe CLI / Vibe Code**, issue de la synthèse de 6 dépôts publics (voir `llm-cli/01-synthese-sources.md`).

- Les skills dans `llm-cli/skills/` suivent le standard Agent Skills : un seul jeu de fichiers sert aux deux CLI (copie vers `~/.gemini/skills/` et `~/.vibe/skills/`).
- Un skill peut embarquer un sous-dossier `scripts/` (Python, bibliothèque standard uniquement) pour des opérations déterministes (ex. `vibe-wise-reset/scripts/reset.py`).
- Les personas `llm-cli/agents/` + `llm-cli/prompts/` sont au format Vibe (TOML + Markdown) : Vibe uniquement.
- Les commandes `llm-cli/gemini/commands/` sont au format TOML de Gemini CLI : Gemini CLI uniquement.
- Procédure d'installation et ordre d'adoption : `llm-cli/03-plan-preparation.md`.
- Toute modification de `llm-cli/` doit rester portable (pas de dépendance à Claude Code, à un orchestrateur externe, ni à des chemins personnels absolus) et en français.

## Ce qu'il ne faut pas faire

- Ne pas modifier la structure sans mettre à jour ce fichier ni le `README.md`.
- Ne pas commiter de secrets (clés API, tokens, `.env`).
- Ne pas rédiger d'article sans validation du plan par le propriétaire.
