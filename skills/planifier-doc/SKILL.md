---
name: planifier-doc
description: Planifie la rédaction d'un article de documentation dans le dépôt docs-tech. Utiliser ce skill AVANT de rédiger, quand l'utilisateur demande de documenter un nouveau sujet ou demande un plan/une préparation d'article. Produit un cadrage (catégorie, audience, scope) et un plan détaillé validé par l'utilisateur.
---

# Planifier la rédaction d'un article

## Quand utiliser ce skill

Toute demande de création d'un nouvel article ou d'une nouvelle cheat sheet. La rédaction elle-même est assurée par le skill `rediger-doc`, uniquement après validation du plan.

## Étapes

### 1. Cadrer la demande

Répondre à ces questions avant tout. Si l'information manque, poser **une seule** question groupée à l'utilisateur :

- **Sujet exact** : quel sujet, et avec quelle délimitation (pas trop large) ?
- **Catégorie** : Guide d'installation | Cheat sheet | Notes & dépannage (cf. `AGENTS.md`).
- **Audience et niveau** : débutant ou avancé ? Le propriétaire lui-même ou partage externe ?
- **Environnement cible** : OS, versions, matériel concernés (ex. Linux, Windows, DIMM DDR4/DDR5).
- **Format souhaité** : pas-à-pas complet ou aide-mémoire condensé.

### 2. Rechercher et vérifier

- Si le sujet implique des faits techniques (versions, commandes, specs), vérifier avec une recherche web plutôt que d'inventer.
- Noter les sources fiables pour la section « Références ».

### 3. Élaborer le plan

Produire un plan avec :

- Titre proposé (clair, orienté tâche ou sujet, en français).
- Objectif en une ou deux phrases.
- Prérequis identifiés.
- Sections prévues (titres H2/H3) avec 1 ligne de description chacune.
- Estimation : cheat sheet (condensé) vs guide complet.
- Nom de dossier proposé (`articles/<kebab-case>/`).

### 4. Faire valider

Présenter le plan à l'utilisateur et attendre son accord explicite avant de passer au skill `rediger-doc`. Ne jamais rédiger sans validation.

## Sortie attendue

Un plan en Markdown, court, présenté dans la conversation (pas encore de fichier créé dans le dépôt à cette étape, sauf demande contraire).
