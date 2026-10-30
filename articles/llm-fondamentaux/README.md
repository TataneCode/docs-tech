# LLM — fondamentaux et panorama

> Catégorie : Guide

**Date :** 2026-10-29
**Statut :** Terminé

## Objectif

Comprendre ce qu'est un grand modèle de langage (LLM), comment il fonctionne et s'entraîne — y compris ce qui se passe à l'intérieur d'un réseau de neurones et lors de l'inférence — pourquoi il hallucine, comment régler sa génération, et savoir choisir entre prompt engineering, fine-tuning, RAG et agents IA — avec un bilan à jour des modèles disponibles en octobre 2026.

## Prérequis

- Aucun prérequis technique : la lecture est progressive, du plus conceptuel au plus opérationnel.
- Pour exécuter les exemples de code : Python 3.10+ et une clé API d'un fournisseur de LLM (les exemples sont fournisseur-neutre quand c'est possible).

## Sommaire des chapitres

| # | Chapitre | Ce que vous saurez faire à la fin |
|---|----------|----------------------------------|
| 1 | [Fonctionnement](01-fonctionnement.md) | Expliquer la prédiction de tokens, l'attention et l'inférence à un collègue |
| 2 | [Entraînement](02-entrainement.md) | Situer le pré-entraînement, le SFT et l'alignement RLHF dans le cycle de vie d'un modèle |
| 3 | [Hallucinations](03-hallucinations.md) | Définir une hallucination, la repérer et réduire le risque |
| 4 | [Température et sampling](04-temperature.md) | Régler température, top-p et top-k selon le cas d'usage |
| 5 | [Prompt engineering](05-prompt-engineering.md) | Rédiger un prompt structuré (rôle, contexte, instruction, format) et itérer |
| 6 | [Fine-tuning et RAG](06-fine-tuning-rag.md) | Choisir entre fine-tuning, RAG et simple prompting selon le besoin |
| 7 | [Créer un agent IA](07-agent-ia.md) | Construire une boucle agent (raisonnement + outils) avec MCP et garde-fous |
| 8 | [Bilan des modèles](08-bilan-modeles.md) | Choisir un modèle (fermé ou open weights) selon coût, latence, confidentialité et usage |
| 9 | [Réseaux de neurones](09-reseaux-de-neurones.md) | Expliquer neurone, couches, propagation avant, rétropropagation et le passage au transformer |
| 10 | [Inférence](10-inference.md) | Comprendre le déroulement de l'inférence : cache KV, quantification, batching, décodage spéculatif, moteurs de serving |

## Ordre de lecture recommandé

1. Chapitres 1, 2 et 9 pour le vocabulaire commun (le chapitre 9 pose les fondations théoriques : réseaux de neurones et entraînement).
2. Chapitre 10 pour comprendre ce qui se passe concrètement quand on appelle un modèle (inférence).
3. Chapitres 3 et 4 pour comprendre les deux comportements qui surprennent toujours : les hallucinations et le hasard contrôlé.
4. Chapitre 5 avant tout projet concret : c'est le levier le plus rapide et le moins coûteux.
5. Chapitre 6 quand le prompting ne suffit plus.
6. Chapitre 7 quand le modèle doit agir, pas seulement répondre.
7. Chapitre 8 pour le choix du modèle, à relire régulièrement : ce chapitre date vite.

## Export PDF

À chaque pull request modifiant ce répertoire, le workflow [`pdf-llm-fondamentaux`](../../../.github/workflows/pdf-llm-fondamentaux.yml) concatène les chapitres 1 à 10 dans l'ordre numérique et génère un PDF (`llm-fondamentaux.pdf`), publié en artefact du run et signalé en commentaire de la PR.

## Pièges connus

- **Les chapitres 1 à 7 et 9-10 sont stables** : les concepts évoluent lentement. Le chapitre 8, lui, reflète l'état du marché en octobre 2026 et doit être relu tous les 2-3 mois.
- **Ne pas confondre « poids ouverts » (open weights) et « open source »** : la plupart des modèles « ouverts » publient leurs poids sous des licences restrictives (voir chapitre 8).
- **Un LLM ne « sait » rien** : il génère du texte plausible. Toute sortie factuelle mérite vérification (chapitre 3).

## Références

- [Hugging Face — documentation LLM](https://huggingface.co/docs/transformers/llm_tutorial) — tutoriels pratiques sur les transformers
- [Lilian Weng — « Large Language Model » (blog)](https://lilianweng.github.io/posts/2023-01-27-the-transformer-family/) — synthèse technique de référence
- [Artificial Analysis — LLM Leaderboard](https://artificialanalysis.ai/leaderboards/models) — classement indépendant des modèles (intelligence, vitesse, prix)
- [3Blue1Brown — « But what is a GPT? » (vidéo)](https://www.3blue1brown.com/lessons/gpt) — visualisation de l'architecture transformer
