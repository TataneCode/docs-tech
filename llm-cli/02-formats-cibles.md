# Formats supportés par les CLI cibles (faits vérifiés)

Objectif : documenter précisément où et comment Gemini CLI et Vibe CLI chargent skills, agents et commandes, pour que l'installation de ce dossier soit exacte.

## Gemini CLI

### Skills (standard Agent Skills)

Gemini CLI supporte nativement le standard [Agent Skills](https://agentskills.io) : un répertoire contenant un `SKILL.md` avec frontmatter YAML.

- **Emplacements** :
  - user : `~/.gemini/skills/<nom>/SKILL.md`
  - projet : `<projet>/.gemini/skills/<nom>/SKILL.md` (ou `.agents/skills/`)
  - extensions packagées
- **Précédence** : user > projet ; dans un même niveau, `.agents/skills/` > `.gemini/skills/`.
- **Frontmatter obligatoire** : `name:` et `description:` doivent être présents, le `---` ouvrant doit être le **tout premier** caractère du fichier (aucun titre, commentaire ou ligne vide avant).
- **Activation** : au démarrage, seul le nom + la description de chaque skill sont injectés ; quand la tâche correspond à la description, Gemini appelle l'outil interne `activate_skill` (avec consentement affiché), puis le corps du `SKILL.md` et le contenu du dossier entrent dans l'historique.
- **Contexte permanent** : fichier `GEMINI.md` (racine projet ou `~/.gemini/GEMINI.md` pour le global).

### Commandes slash custom (TOML)

- **Emplacements** : `~/.gemini/commands/` (user) et `<projet>/.gemini/commands/` (projet).
- **Format** : un fichier `.toml` par commande. Le chemin relatif devient le nom de la commande ; un sous-dossier crée un namespace (`commands/git/commit.toml` → `/git:commit`).
- **Champs** :
  - `prompt` (string, requis) : le prompt envoyé au modèle. Multi-ligne via `"""..."""`. `{{args}}` est remplacé par les arguments saisis.
  - `description` (string, optionnel) : affiché dans l'aide.
  - `!{...}` dans le prompt : exécute une commande shell et injecte sa sortie.
  - `@{chemin}` dans le prompt : injecte le contenu d'un fichier.
- **Gestion** : `/commands list` pour lister, `/commands reload` pour recharger sans redémarrer.

### Agents / sous-agents

Gemini CLI n'a pas de mécanisme de « personas » déclarés dans un fichier de config. Les rôles spécialisés passent par :

- des **skills** (activation par description), ou
- des **commandes TOML** qui définissent le rôle dans le prompt (c'est le choix retenu ici), ou
- des extensions / MCP.

## Vibe CLI / Vibe Code (Mistral)

Vibe suit la spécification Agent Skills pour les skills : les mêmes `SKILL.md` fonctionnent tels quels.

### Skills

- **Emplacements** :
  - user : `~/.vibe/skills/<nom>/SKILL.md` (ou `$VIBE_HOME/skills/` si `VIBE_HOME` est défini)
  - projet : `./.vibe/skills/<nom>/SKILL.md` (dossier de confiance requis)
  - chemins custom : option `skill_paths` dans `~/.vibe/config.toml`
- **Frontmatter** : `name`, `description` ; options supportées : `user-invocable` (défaut `true` → la skill devient une commande slash `/<nom>` avec autocomplétion), `disable-model-invocation` (l'agent ne la charge pas seul), `allowed-tools` (lu mais non imposé).
- **Activation** : par slash command explicite, ou automatiquement quand la description correspond à la tâche.

### Agents (personas)

- **Emplacements** : `~/.vibe/agents/*.toml` (user) et `./.vibe/agents/*.toml` (projet, dossier de confiance).
- **Format TOML** (champs utiles) :

```toml
agent_type = "subagent"
display_name = "Code Reviewer"
description = "Revue de code : qualité, sécurité, performance."
system_prompt_id = "code-reviewer"   # fichier ~/.vibe/prompts/code-reviewer.md
safety = "safe"                       # ou "destructive"
enabled_tools = ["read_file", "grep", "glob", "bash"]
```

- Le `system_prompt_id` référence un fichier Markdown dans `./.vibe/prompts/` (projet) ou `~/.vibe/prompts/` (user).
- Les agents built-in (ask, plan, code-explorer...) servent de référence de style.
- **Contexte permanent** : `AGENTS.md` (racine projet ou `~/.vibe/AGENTS.md` pour le user-level), lu automatiquement.

### Hooks, plugins

- Hooks et plugins (`~/.vibe/plugins/`) existent mais ne sont **pas retenus** ici : les hooks « always-on » d'i-have-adhd sont spécifiques à d'autres hôtes et seront remplacés par un `AGENTS.md` user-level (voir plan).

## Tableau récapitulatif d'installation

| Élément du dossier `llm-cli/` | Gemini CLI | Vibe CLI |
|-------------------------------|------------|----------|
| `skills/<nom>/SKILL.md` | `cp -r` → `~/.gemini/skills/` | `cp -r` → `~/.vibe/skills/` |
| `agents/*.toml` | — | `cp` → `~/.vibe/agents/` |
| `prompts/*.md` | — | `cp` → `~/.vibe/prompts/` |
| `gemini/commands/*.toml` | `cp` → `~/.gemini/commands/` | — |
| Règles Karpathy + style TDAH (always-on) | `~/.gemini/GEMINI.md` | `~/.vibe/AGENTS.md` |

## Pièges connus

- **Frontmatter SKILL.md** : un seul caractère avant le `---` ouvrant et le skill est ignoré silencciosement (Gemini comme Vibe). Toujours commencer le fichier par `---`.
- **Nom de skill** : le nom affiché vient du frontmatter `name`, pas du dossier — les garder identiques évite les surprises.
- **Confiance dossier (Vibe)** : les skills/agents locaux (`.vibe/`) ne sont chargés que dans un dossier marqué « trusted » ; les versions user (`~/.vibe/`) n'ont pas cette contrainte.
- **Commandes Gemini** : le nom de la commande est dérivé du chemin de fichier, casse comprise ; pas d'espace dans les noms de fichier.
- **Ne pas installer les deux mécanismes « always-on »** : si `AGENTS.md`/`GEMINI.md` global rend le style TDAH permanent, le skill `i-have-adhd` reste utile en invocation explicite (`/i-have-adhd`) pour les sessions où le style par défaut a été écrasé.

## Références

- [Gemini CLI — Agent Skills](https://geminicli.com/docs/cli/skills/) · [Tutoriel skills](https://geminicli.com/docs/cli/tutorials/skills-getting-started/)
- [Gemini CLI — Commandes custom (TOML)](https://geminicli.com/docs/cli/custom-commands/)
- [Mistral Vibe — Skills](https://docs.mistral.ai/vibe/code/cli/skills) · [Agents](https://docs.mistral.ai/vibe/code/cli/agents)
- [Spécification Agent Skills](https://agentskills.io)
- Dépôt Vibe (code source) : [mistralai/mistral-vibe](https://github.com/mistralai/mistral-vibe)
