# Chapitre 1 — Fonctionnement d'un LLM

> Partie de : [LLM — fondamentaux et panorama](README.md)

**Date :** 2026-10-29

## Objectif

Expliquer, sans formule mathématique, comment un grand modèle de langage produit du texte : la prédiction de tokens, l'architecture transformer, le rôle du contexte et les limites de l'inférence.

## Qu'est-ce qu'un LLM

Un **grand modèle de langage** (Large Language Model, LLM) est un réseau de neurones entraîné à prédire le mot suivant (plus précisément : le **token** suivant) dans un texte. C'est tout. Toute l'apparente « intelligence » découle de cette tâche simple, poussée à une échelle massive :

- **Des milliards de paramètres** (poids internes du réseau), ajustés pendant l'entraînement.
- **Des centaines de milliards de tokens** vus pendant l'entraînement (livres, code, pages web, conversations).
- **Une architecture « transformer »** qui permet de traiter de longs textes en parallèle et de relier des mots éloignés.

## Le token, unité de base

Le texte n'est pas traité lettre par lettre. Il est découpé en **tokens** : morceaux de mots fréquents (un token ≈ 3-4 caractères en français, un peu moins en anglais).

```
"Le chat dort."  →  ["Le", " chat", " dort", "."]   →  4 tokens
```

Pourquoi cela compte :

- **Le coût** : les API facturent au token (entrée + sortie).
- **La fenêtre de contexte** : un modèle accepte un nombre maximum de tokens en entrée (de quelques milliers à plus d'un million selon les modèles, voir chapitre 8).
- **La latence** : générer 500 tokens prend plus de temps que 50.

## La prédiction de tokens : le cœur du système

À chaque étape, le modèle reçoit une séquence de tokens et produit une **distribution de probabilité sur tout le vocabulaire** (~50 000 à 200 000 tokens possibles). Il choisit le token suivant selon cette distribution, l'ajoute à la séquence, et recommence.

```
Entrée :  "La capitale de la France est"
Étape 1 :  proba(" Paris") = 0,91  →  on ajoute " Paris"
Entrée :  "La capitale de la France est Paris"
Étape 2 :  proba(".")     = 0,87  →  on ajoute "."
...
```

La génération est **autorégressive** : chaque nouveau token dépend de tous les précédents. C'est pourquoi l'ordre des mots compte et pourquoi le modèle « avance » de gauche à droite.

## L'attention : relier les mots entre eux

Le mécanisme central du transformer s'appelle **l'attention** (self-attention). Pour chaque token, le modèle calcule à quels tokens précédents il doit « prêter attention ».

Exemple :

```
"Marie a posé son sac. Elle est partie."
              ↑
        "Elle" prête attention à "Marie" (résolution de la coréférence)
```

L'attention permet de :

- **Relier des informations éloignées** dans un long texte.
- **Pondérer** ce qui est pertinent pour prédire le token suivant.
- **Traiter le texte en parallèle** pendant l'entraînement (contrairement aux anciens modèles récurrents).

Un LLM moderne empile des dizaines de **couches** (typiquement 32 à 128), chacune avec plusieurs « têtes » d'attention qui observent le texte sous différents angles (syntaxe, coréférence, sémantique, position…).

## De l'entraînement à l'inférence

Deux phases distinctes :

| Phase | Nom | Ce qui se passe |
|-------|-----|-----------------|
| Entraînement | « Training » | Les poids du réseau sont ajustés sur des milliards de tokens. Coûteux (jours à mois sur des milliers de GPU). Fait une seule fois par le laboratoire (voir chapitre 2). |
| Inférence | « Serving » | Les poids sont figés. Le modèle génère du texte, token par token, sur votre requête. C'est ce que vous payez quand vous appelez une API. |

**Conséquence pratique** : vous ne pouvez pas « apprendre » quelque chose à un modèle pendant une conversation au-delà de ce qui tient dans la fenêtre de contexte. Le modèle n'a pas de mémoire persistante entre deux appels (sauf si l'application la construit elle-même).

## La fenêtre de contexte

La **fenêtre de contexte** (context window) est le nombre maximum de tokens que le modèle peut « voir » en même temps : prompt système + historique de conversation + documents + réponse en cours de génération.

- Tout ce qui dépasse est tronqué ou perdu.
- Un contexte long coûte plus cher et ralentit la génération.
- Les modèles récents atteignent 128 000 à 1 000 000+ de tokens ; les plus anciens se limitent à 4 000-8 000.

**Piège** : un modèle avec une grande fenêtre n'utilise pas nécessairement bien tout ce contexte. La pertinence décroît souvent avec la distance (« lost in the middle » : l'information au milieu d'un long contexte est moins bien utilisée que celle au début ou à la fin).

## Ce que le modèle « comprend » (et ne comprend pas)

Le modèle manipule des **régularités statistiques** apprises sur son corpus d'entraînement. Il n'a pas de représentation du monde, pas de mémoire propre, pas de « bon sens » au sens humain.

| Le modèle fait bien | Le modèle ne fait pas |
|---------------------|----------------------|
| Générer du texte fluide et contextuellement adapté | « Savoir » des faits avec certitude |
| Suivre des instructions, changer de registre, traduire | Accéder à Internet ou à vos fichiers (sauf outillage spécifique) |
| Raisonner sur du texte fourni dans le contexte | Se souvenir d'une conversation précédente (sans mémoire externe) |
| Reconnaître des patterns (code, tableaux, formats) | Garantir une réponse correcte ou à jour |

## Architecture en deux familles

Deux grandes familles coexistent en 2026 :

- **Dense** : tous les paramètres sont actifs pour chaque token. Simples, efficaces à petite taille (ex. : Llama, Gemma, Mistral en versions compactes).
- **Mixture-of-Experts (MoE)** : le réseau contient plusieurs « experts » spécialisés ; un routeur n'active que quelques experts par token (ex. : DeepSeek V4, GLM-5.x, Kimi, Qwen3 en versions MoE). Résultat : un modèle de centaines de milliards de paramètres « totaux » mais seulement quelques dizaines de milliards actifs par token — meilleur rapport qualité/coût de calcul.

## Inférence : les optimisations qui rendent le service abordable

Faire tourner un LLM en production demande des optimisations, sinon le coût explose :

- **Quantification** : les poids sont stockés en précision réduite (FP16, INT8, INT4, voire 1-2 bits). Un modèle de 70 milliards de paramètres tient ainsi sur 1-2 GPU au lieu de 4-8. Léger sacrifice de qualité, énorme gain de coût.
- **Cache KV** : les étapes d'attention pour les tokens déjà traités sont mises en cache, évitant de les recalculer à chaque nouveau token.
- **Batching continu** et **spéculative decoding** (un petit modèle « brouillon » propose plusieurs tokens, le grand modèle les vérifie en un seul passage) : accélèrent la génération.
- **Serving local** : des moteurs comme llama.cpp, Ollama, vLLM ou LM Studio font tourner des modèles sur votre propre machine (voir l'article [LM Studio sur Ubuntu](../lm-studio-ubuntu/README.md)).

## Pièges connus

- **« Le modèle a été entraîné sur X donc il sait X »** : faux. L'entraînement crée des patterns, pas une base de connaissances fiable (voir chapitre 3 sur les hallucinations).
- **Confondre taille et qualité** : un modèle plus gros n'est pas automatiquement meilleur pour votre tâche ; un petit modèle bien choisi (ou fine-tuné) peut suffiser.
- **Sous-estimer le coût au token** : un prompt système de 5 000 tokens envoyé à chaque requête coûte 5 000 tokens à chaque appel, même si la réponse fait 20 tokens.

## Références

- [3Blue1Brown — « But what is a GPT? »](https://www.3blue1brown.com/lessons/gpt) — visualisation animée de l'architecture (en anglais, accessible)
- [Lilian Weng — « The Transformer Family »](https://lilianweng.github.io/posts/2023-01-27-the-transformer-family/) — synthèse technique complète (en anglais)
- [Hugging Face — « LLM Tutorial »](https://huggingface.co/docs/transformers/llm_tutorial) — mise en pratique avec des modèles open weights
- [Andrej Karpathy — « Intro to Large Language Models » (vidéo, 1h)](https://www.youtube.com/watch?v=zjkBMFhNj_g) — conférence de référence (en anglais)
- [colibri — moteur d'inférence MoE en C pur](../colibri/README.md) — exemple concret d'optimisations d'inférence (VRAM/RAM/disque, cache KV, MTP)
