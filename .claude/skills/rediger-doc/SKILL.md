---
name: rediger-doc
description: Rédige un article de documentation conforme aux conventions du dépôt docs-tech (template, style, structure). Utiliser ce skill après validation du plan (skill planifier-doc) pour écrire un nouvel article ou réécrire substantiellement un article existant.
---

# Rédiger un article de documentation

## Prérequis

- Le plan a été validé par l'utilisateur (skill `planifier-doc`).
- Lire `AGENTS.md` et `_templates/article-template.md` avant d'écrire.

## Étapes

### 1. Préparer le fichier

- Créer le dossier `articles/<nom-kebab-case>/` et copier la structure du template dans `README.md`.
- Remplir l'en-tête : catégorie, date (YYYY-MM-DD), statut (`Brouillon` jusqu'à validation, puis `Terminé`).

### 2. Rédiger

Règles impératives (cf. `AGENTS.md` — Style de rédaction) :

- Français, ton direct et factuel, sans remplissage.
- Ordre du template : **Objectif → Prérequis → Contenu (sections/étapes) → Pièges connus → Références**.
- Commandes dans des blocs de code annotés (```bash, ```powershell...) et **testables** : indiquer l'OS/le contexte quand il compte.
- Chemins, commandes, noms de fichiers en `code inline`.
- Markdown pur, pas d'HTML.
- Pas de commentaires de code superflus dans les blocs, seulement les nécessaires à la compréhension.
- Chaque commande ou étape doit être vérifiable : si un fait n'est pas sûr, le vérifier (recherche web ou exécution locale si possible) plutôt que d'inventer.
- Les cheat sheets privilégient tableaux et listes condensées ; les guides privilégient le pas-à-pas numéroté.

### 3. Finaliser

- Mettre à jour le **sommaire du `README.md` racine** (tableau : article, catégorie, description).
- Vérifier la cohérence des liens internes (chemins relatifs) et externes (liens Markdown descriptifs).

### 4. Livrer

- Créer une branche dédiée (pas de push direct sur `main`), committer en français (impératif présent, une ligne concise).
- Ouvrir une draft PR avec un résumé des changements et les vérifications faites.
- Signaler l'URL de la PR à l'utilisateur et demander si le suivi CI est souhaité.

## Auto-vérification avant commit

- [ ] L'article suit l'ordre du template.
- [ ] Toutes les commandes ont leur contexte (OS, version, prérequis).
- [ ] Le sommaire racine est à jour.
- [ ] Aucun secret (clé, token, mot de passe) dans le contenu.
- [ ] Statut et date corrects dans l'en-tête.
