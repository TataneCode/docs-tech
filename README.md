# Docs Tech

Documentation centralisée de sujets techniques divers : guides d'installation, cheat sheets, notes de configuration et de dépannage.

## Sommaire

| Article | Catégorie | Description |
|---------|-----------|-------------|
| [Héberger des LLM légers avec LM Studio sur un ThinkPad T14 sous Ubuntu](articles/lm-studio-ubuntu/README.md) | Guide d'installation | Installation de LM Studio sur Ubuntu, choix de modèles Qwen pour 32 Go en CPU, serveur API local et déploiement Docker |

## Structure du dépôt

```
docs-tech/
├── README.md            # Ce fichier : accueil et sommaire
├── _templates/          # Templates à copier pour un nouvel article
└── articles/            # Un dossier par sujet
    └── <sujet>/README.md
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
