# llm-cli

Boîte à outils personnelle pour agents CLI LLM : **skills**, **agents** (personas) et **commandes slash** prêts à l'emploi pour **Gemini CLI** et **Vibe CLI / Vibe Code** (Mistral).

## Objectif

Capitaliser les meilleures pratiques de la communauté (workflows agent, discipline de code, style de sortie) dans un format portable, puis les installer en quelques commandes sur chaque CLI.

## Contenu

```
llm-cli/
├── README.md                    # Ce fichier
├── 01-synthese-sources.md       # Synthèse des 4 repos étudiés (skills, agents, commandes)
├── 02-formats-cibles.md         # Formats supportés par Gemini CLI et Vibe CLI (faits vérifiés)
├── 03-plan-preparation.md       # Plan de préparation et d'installation, étape par étape
├── skills/                      # Skills au standard Agent Skills (SKILL.md, YAML frontmatter)
│   └── <nom-du-skill>/SKILL.md  # Portable : Gemini CLI, Vibe, Claude Code, Codex...
├── agents/                      # Personas Vibe (TOML, ~/.vibe/agents/)
├── prompts/                     # Prompts système des personas (Markdown, ~/.vibe/prompts/)
└── gemini/commands/             # Commandes slash Gemini CLI (TOML, ~/.gemini/commands/)
```

## Les 18 skills retenus

| Skill | Source | Utilité principale |
|-------|--------|--------------------|
| `karpathy-guidelines` | andrej-karpathy-skills | Règles de comportement : simplicité, changements chirurgicaux, critères de succès vérifiables |
| `i-have-adhd` | i-have-adhd | Style de sortie adapté TDAH : action d'abord, étapes numérotées, pas de remplissage |
| `verification-gate` | superpowers + claude-octopus | Interdiction de prétendre que ça marche sans preuve exécutée |
| `systematic-debugging` | superpowers | Debug en 4 phases : cause racine avant correctif |
| `tdd` | superpowers + claude-octopus | Red-Green-Refactor, test d'abord |
| `writing-plans` | superpowers + claude-octopus | Plan structuré avant de toucher au code |
| `work-slicing` | claude-octopus | Découper un plan en tranches verticales fileables |
| `brainstorming` | superpowers + claude-octopus | Cadrer un problème avant de proposer une solution |
| `code-review` | superpowers + claude-octopus | Demander/recevoir une revue de code rigoureuse |
| `finish-branch` | superpowers + claude-octopus | Terminer proprement une branche (tests, merge ou PR) |
| `prd` | claude-octopus | Rédiger un PRD après clarification obligatoire |
| `thought-partner` | claude-octopus | Session de réflexion guidée (4 techniques de percée) |
| `security-audit` | claude-octopus | Audit sécurité OWASP, version mono-modèle |
| `parallel-agents` | superpowers + claude-octopus | Dispatcher des tâches indépendantes en parallèle |
| `feynman-learning` | OpenMAIC | Apprentissage par l'explication : teach-back, trou minimal, questions socratiques, transfer |
| `deep-research` | OpenMAIC | Recherche web budgétée avant rédaction, avec registre claim → source |
| `fact-check` | OpenMAIC | Vérification factuelle ciblée d'un contenu existant (erreurs avérées, formulations trompeuses) |
| `curriculum-planner` | OpenMAIC | Planifier une série de leçons cohérentes, avec validation de la liste complète avant production |

## Installation rapide

Voir [03-plan-preparation.md](03-plan-preparation.md). En résumé :

```bash
# Skills (identiques pour Gemini CLI et Vibe)
cp -r llm-cli/skills/* ~/.gemini/skills/    # Gemini CLI (user scope)
cp -r llm-cli/skills/* ~/.vibe/skills/      # Vibe CLI / Vibe Code

# Personas (Vibe uniquement)
cp llm-cli/agents/*.toml ~/.vibe/agents/
cp llm-cli/prompts/*.md ~/.vibe/prompts/

# Commandes slash (Gemini CLI uniquement)
cp -r llm-cli/gemini/commands/* ~/.gemini/commands/
```

## Provenance

- [obra/superpowers](https://github.com/obra/superpowers) — 14 skills de discipline de développement
- [multica-ai/andrej-karpathy-skills](https://github.com/multica-ai/andrej-karpathy-skills) — guidelines comportementales (MIT)
- [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd) — skill de style de sortie TDAH (MIT)
- [nyldn/claude-octopus](https://github.com/nyldn/claude-octopus) — 63 skills, 54 commandes, 31 personas orientés multi-LLM
- [THU-MAIC/OpenMAIC](https://github.com/THU-MAIC/OpenMAIC) — 25 skills de pédagogie agentique (MIT) : 4 retenus, portés en version générique

Tous les contenus portés ici sont des adaptations en français ; les mécanismes dépendant de Claude Octopus (orchestration multi-fournisseurs), de Claude Code ou du runtime OpenMAIC (outils `create_stage`, `generate_scene`, DSL de slides) ont été retirés ou remplacés par des équivalents génériques.
