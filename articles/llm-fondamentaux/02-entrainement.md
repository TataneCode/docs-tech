# Chapitre 2 — L'entraînement d'un LLM

> Partie de : [LLM — fondamentaux et panorama](README.md)

**Date :** 2026-10-29

## Objectif

Comprendre les trois grandes phases qui mènent d'un modèle aléatoire à un assistant utile — pré-entraînement, fine-tuning (SFT) et alignement — et ce que chacune apporte (et coûte).

## Vue d'ensemble

L'entraînement d'un LLM moderne se déroule en plusieurs phases successives. Chacune répond à une question différente :

```
Phase 1 : Pré-entraînement (self-supervised)
    →  Le modèle apprend la langue, les faits du monde, le raisonnement de base
    →  « Lire tout Internet »

Phase 2 : Fine-tuning supervisé (SFT — Supervised Fine-Tuning)
    →  Le modèle apprend à suivre des instructions et à converser
    →  « Apprendre le format question → réponse utile »

Phase 3 : Alignement (RLHF / DPO / constitutionnel)
    →  Le modèle apprend les préférences humaines : ton, refus, style
    →  « Devenir agréable et sûr à utiliser »

Phase 4 (optionnelle) : Post-entraînement spécialisé
    →  Raisonnement renforcé, code, tool use, domaines métier
    →  « Devenir expert sur une tâche précise »
```

## Phase 1 — Le pré-entraînement

**Objectif** : apprendre à prédire le token suivant sur une masse énorme de texte.

- **Données** : pages web (Common Crawl), livres, code source (GitHub), articles scientifiques, forums… Des billions, voire dizaines de billions de tokens. Le filtrage qualité est déterminant : un corpus mieux filtré donne un meilleur modèle à taille égale.
- **Méthode** : apprentissage auto-supervisé — pas de labels humains, le texte lui-même fournit la réponse (le token suivant).
- **Coût** : c'est la phase la plus chère. Les modèles frontier de 2026 ont été entraînés sur des dizaines de milliers de GPU pendant des semaines. C'est pourquoi seuls quelques laboratoires (OpenAI, Anthropic, Google, Meta, DeepSeek, Alibaba, Moonshot, Z.ai, Mistral, xAI, Xiaomi…) entraînent leurs propres modèles de base.
- **Ce que le modèle « emmagasine »** : des statistiques sur la langue, des connaissances factuelles (mémorisables, parfois de façon imparfaite), des patterns de raisonnement, du code, des langues étrangères.

**Limite fondamentale** : le modèle apprend des corrélations, pas des vérités. Une information fausse mais fréquente dans le corpus peut être « apprise » aussi sûrement qu'une vraie (d'où les hallucinations, chapitre 3).

## Phase 2 — Le fine-tuning supervisé (SFT)

Un modèle pré-entraîné « complète » du texte : il n'est pas encore un assistant. Le SFT lui apprend le **format conversation** :

- **Données** : des centaines de milliers à quelques millions de paires (instruction, réponse idéale), rédigées par des humains ou générées par des modèles plus forts puis vérifiées. On y trouve aussi des exemples de refus, de clarification, de suivi de conversation multi-tours.
- **Méthode** : même objectif de prédiction du token suivant, mais sur ces données d'instructions. Les poids du modèle continuent d'être ajustés (fine-tuning **complet** ou **partiel** via LoRA, voir chapitre 6).
- **Effet** : le modèle passe de « compléteur de texte » à « assistant qui répond à une question ».

## Phase 3 — L'alignement

Le SFT ne suffit pas : un modèle peut être utile mais désagréable, verbeux, enclin à fabriquer des réponses, ou dangereux sur certains sujets. L'alignement corrige cela.

Deux grandes familles de méthodes :

- **RLHF (Reinforcement Learning from Human Feedback)** : des humains comparent deux réponses du modèle ; un « modèle de récompense » apprend ces préférences ; le LLM est ajusté par renforcement pour maximiser la récompense.
- **DPO (Direct Preference Optimization)** et dérivés : plus simple et stable que le RLHF, très utilisé aujourd'hui. Le modèle apprend directement des paires « réponse préférée / réponse rejetée » sans modèle de récompense intermédiaire.
- **Approches constitutionnelles** : le modèle s'auto-évalue contre un ensemble de principes écrits (ex. : « ne pas aider à faire du mal »), réduisant le besoin de labels humains massifs.

**Ce que l'alignement change concrètement** : le ton, la concision, la capacité à dire « je ne sais pas », les refus sur les sujets sensibles, le respect du format demandé.

**Piège** : l'alignement peut aussi rendre le modèle **trop prudent** (refus excessifs) ou **trop lisse** (réponses génériques). Les modèles « de base » (pré-entraînés uniquement) restent recherchés pour certains usages techniques car ils sont moins « policés ».

## Phase 4 — Le post-entraînement spécialisé

Pour pousser un modèle sur une tâche précise, les laboratoires ajoutent des phases ciblées :

- **Raisonnement étendu** : entraînement sur des problèmes de maths, logique, code, avec de longues chaînes de pensée (« extended thinking », « chain-of-thought »). Coûteux en calcul d'inférence mais très efficace sur les tâches difficiles.
- **Tool use / function calling** : entraînement à décider quand appeler un outil externe (recherche web, calculatrice, API) et à formater correctement l'appel.
- **Contexte long** : entraînement spécifique pour utiliser efficacement des centaines de milliers de tokens.
- **Multimodal** : ajout de capacités vision (images), audio, vidéo en entrée.

## Le cycle de vie complet, résumé

```
Données brutes (billions de tokens)
        │
        ▼
Pré-entraînement  ──►  modèle de base (base model)
        │                 « compléteur de texte savant »
        ▼
SFT (instructions)  ──►  modèle instruct
        │                 « assistant qui suit les instructions »
        ▼
Alignement (RLHF/DPO)  ──►  modèle aligné
        │                     « assistant utile, poli, sûr »
        ▼
Post-entraînement  ──►  modèle spécialisé
                           « expert raisonnement / code / tool use »
        │
        ▼
   Déploiement (API ou local)  ──►  votre application
```

## Qui fait quoi, en pratique

| Rôle | Acteur | Ce qu'il fait |
|------|-------|---------------|
| Entraîne le modèle de base | Laboratoire (OpenAI, Anthropic, Google, Meta, DeepSeek, Alibaba, Mistral, Z.ai, Moonshot, xAI, Xiaomi…) | Phases 1 à 4. Coût : millions à centaines de millions d'euros. |
| Fine-tune pour un usage métier | Votre équipe, ou prestataire | Phase 2/4 sur vos données (voir chapitre 6). Coût : de quelques centaines d'euros (LoRA sur GPU cloud) à des dizaines de milliers. |
| Utilise le modèle | Vous | Inférence + prompt engineering (chapitre 5). Coût : au token, ou matériel local. |

## Pourquoi il y a autant de « versions » d'un même modèle

Un laboratoire publie souvent plusieurs variantes d'une même famille :

- **Base** vs **Instruct/Chat** : avec ou sans phases 2-3.
- **Tailles** : 3B, 8B, 70B, 400B… (même entraînement, échelles différentes).
- **Quantized** : le même modèle en INT8/INT4 pour tourner sur moins de mémoire.
- **Raisonnement** (« R1 », « thinking », « Max ») : avec post-entraînement raisonnement renforcé.
- **Spécialisés** : code (Coder), vision (VL), agents, voix.

## Pièges connus

- **« L'entraînement rend le modèle factuel »** : non. L'entraînement rend le modèle *plausible*. La fraîcheur des connaissances est limitée par la date de coupure du corpus (souvent plusieurs mois avant la sortie du modèle).
- **Sous-estimer le coût du pré-entraînement** : c'est la raison d'être des modèles open weights — réutiliser l'entraînement des autres plutôt que le refaire (chapitres 6 et 8).
- **Négliger la qualité des données de SFT** : quelques milliers d'exemples bien écrits battent des millions d'exemples médiocres. C'est le nerf de la guerre du fine-tuning.

## Références

- [Hugging Face — « Fine-tuning »](https://huggingface.co/docs/transformers/training) — guide pratique du fine-tuning
- [Hugging Face — « The Ultra-Scale Playbook »](https://huggingface.co/spaces/HuggingFaceH4/blog-posts/blob/main/ultra-scale-playbook.md) — retour d'expérience sur l'entraînement à grande échelle (en anglais)
- [Anthropic — « Core Views on AI Safety » / billets de recherche](https://www.anthropic.com/research) — réflexions sur l'alignement (en anglais)
- [Stanford CRFM — « Holistic Evaluation of Language Models » (HELM)](https://crfm.stanford.edu/helm/latest/) — méthodologie d'évaluation des modèles (en anglais)
- [DeepSeek — article R1 (arXiv)](https://arxiv.org/abs/2501.12948) — exemple de post-entraînement raisonnement par renforcement (en anglais)
