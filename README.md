# Docs Tech

Documentation centralisée de sujets techniques divers : guides d'installation, cheat sheets, notes de configuration et de dépannage.

## Sommaire

| Article | Catégorie | Description |
|---------|-----------|-------------|
| [Héberger des LLM légers avec LM Studio sur un ThinkPad T14 sous Ubuntu](articles/lm-studio-ubuntu/README.md) | Guide d'installation | Installation de LM Studio sur Ubuntu, choix de modèles Qwen pour 32 Go en CPU, serveur API local et déploiement Docker |
| [Shell Linux — cheat sheet développeur](articles/linux-shell/README.md) | Cheat sheet | Commandes shell du quotidien : permissions, grep dans les logs, redirections, processus |
| [Vim — cheat sheet développeur](articles/vim/README.md) | Cheat sheet | Consulter et éditer fichiers et code : modes, navigation, édition, recherche, config minimale |
| [llm-cli — skills, agents et commandes pour Gemini CLI et Vibe](llm-cli/README.md) | Cheat sheet | Boîte à outils personnelle : 20 skills portables, 7 personas Vibe, 8 commandes slash Gemini, synthèse de 6 dépôts sources |
| [colibri — synthèse et mise en place de GLM-5.3-Flash](articles/colibri/README.md) | Guide d'installation | Moteur d'inférence C pur pour modèles MoE géants sur matériel ordinaire : fonctionnement, installation de GLM-5.3-Flash, analyse d'un fork Rust |
| [FreeLLMAPI — agréger les free tiers LLM et connecter OpenCode](articles/freellmapi/README.md) | Guide d'installation | Routeur local OpenAI-compatible agrégeant 34 free tiers de fournisseurs LLM : installation Docker, clé API unifiée et connexion d'OpenCode via un provider custom dans `opencode.json` |
| [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md) | Guide | Série de 8 chapitres sur les LLM en français : fonctionnement, entraînement, hallucinations, température, prompt engineering, fine-tuning/RAG, création d'un agent IA et bilan des modèles (octobre 2026) |

## Structure du dépôt

```
docs-tech/
├── README.md            # Ce fichier : accueil et sommaire
├── AGENTS.md            # Contexte projet
├── _templates/          # Templates à copier pour un nouvel article
├── articles/            # Un dossier par sujet
│   └── <sujet>/README.md
├── skills/              # Skills du projet (planifier-doc, rediger-doc)
└── llm-cli/             # Skills, agents et commandes pour Gemini CLI et Vibe
    ├── 01-synthese-sources.md
    ├── 02-formats-cibles.md
    ├── 03-plan-preparation.md
    ├── skills/          # 20 skills au standard Agent Skills (SKILL.md)
    ├── agents/          # 7 personas Vibe (TOML)
    ├── prompts/         # Prompts système des personas (Markdown)
    └── gemini/commands/ # 8 commandes slash Gemini CLI (TOML)
```

## Catégories

- **Guides d'installation** : procédures pas à pas (ex. GLM Studio)
- **Cheat sheets** : aide-mémoire condensé (ex. DIMM, Linux)
- **Notes & dépannage** : problèmes rencontrés et leurs solutions

## Comment ajouter un article

1. Créer un dossier `articles/<nom-du-sujet>/`
2. Copier `_templates/article-template.md` vers `articles/<nom-du-sujet>/README.md`
3. Rédiger l'article en suivant le template
4. Ajouter une ligne dans le sommaire ci-dessus
