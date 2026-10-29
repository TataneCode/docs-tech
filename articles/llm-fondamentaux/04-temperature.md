# Chapitre 4 — Température et paramètres de génération

> Partie de : [LLM — fondamentaux et panorama](README.md)

**Date :** 2026-10-29

## Objectif

Comprendre ce que règlent la température, top-p et top-k (et les autres paramètres de sampling), et savoir quelles valeurs choisir selon le cas d'usage.

## Le problème : le hasard de la génération

Rappel (chapitre 1) : à chaque étape, le modèle produit une **distribution de probabilité** sur le vocabulaire. Il faut choisir un token. Deux extrêmes :

- **Choisir toujours le plus probable** → génération déterministe, mais répétitive, rigide.
- **Tirer au sort selon les probabilités** → génération variée, mais parfois incohérente.

Les paramètres de sampling règlent ce curseur entre **fidélité** et **créativité**.

## La température

La température **T** redresse ou aplati la distribution de probabilité avant le tirage.

Formellement : `proba ajustée(token) = proba(token) ^ (1/T)` (puis renormalisation).

| Température | Effet | Usage typique |
|-------------|-------|---------------|
| **0** | Toujours le token le plus probable (déterministe) | Extraction de données, classification, formatage structuré, réponses factuelles courtes |
| **0,1 – 0,3** | Très concentré, quasi déterministe | RAG, résumés factuels, code, traduction technique |
| **0,5 – 0,8** | Équilibré : fidèle avec une légère variété | Usage général, conversation, rédaction assistée |
| **1,0** | Distribution « naturelle » du modèle | Brainstorming, rédaction créative, génération d'idées |
| **> 1,0** (1,2 – 2,0) | Distribution aplatie, très aléatoire | Créativité extrême, génération de variantes — au risque de l'incohérence |

**Règle empirique** : si la sortie doit être **vérifiable** (fait, chiffre, code, JSON), température ≤ 0,3. Si la sortie doit être **variée** (idées, brouillons, style), 0,7-1,0.

**Piège** : la plupart des API exposent `temperature` entre 0 et 2, mais certains fournisseurs la limitent à 0-1. Vérifier la documentation du fournisseur.

## Top-p (nucleus sampling)

Au lieu de tirer dans tout le vocabulaire, on ne considère que les tokens les plus probables dont la **somme des probabilités atteint p**.

```
Distribution :  "Paris" 0,60   "Lyon" 0,15   "Marseille" 0,08   … (reste) 0,17
top-p = 0,9  →  on garde "Paris" + "Lyon" + "Marseille" (0,60+0,15+0,08 = 0,83…
                on ajoute le suivant jusqu'à ≥ 0,90), le reste est exclu.
```

| top-p | Effet |
|-------|-------|
| **0,1 – 0,5** | Très restrictif : on pioche parmi les 1-3 meilleurs candidats |
| **0,9 – 0,95** | Standard recommandé : qualité maintenue, variété préservée |
| **1,0** | Aucun filtrage (tout le vocabulaire est éligible) |

**top-p est souvent préférable à la température seule** : il s'adapte à la « certitude » du modèle. Quand le modèle est sûr (un seul token domine), top-p ne change rien. Quand il hésite, top-p élimine la « longue traîne » improbable.

## Top-k

On ne garde que les **k tokens les plus probables**, quoi qu'il arrive.

| top-k | Effet |
|-------|-------|
| **1** | Équivalent à température 0 (toujours le meilleur) |
| **10 – 50** | Élimine les candidats absurdes tout en gardant de la variété |
| **0 ou absent** | Désactivé |

Moins utilisé que top-p aujourd'hui, mais encore présent dans certaines API et moteurs locaux.

## Combiner les paramètres : la recette standard

La pratique courante (et le défaut de nombreuses API) :

```
temperature = 0,7   (ou 0 pour les tâches factuelles)
top_p       = 0,9   (ou 0,95)
top_k       = désactivé (ou 40)
```

**Ne jamais régler les trois en même temps à des valeurs agressives** : température haute + top-p bas est contradictoire et donne des résultats imprévisibles. Règle : **un seul levier principal** (souvent top-p, ou température), les autres à des valeurs neutres.

## Les autres paramètres utiles

| Paramètre | Rôle | Conseil |
|-----------|------|---------|
| `max_tokens` | Nombre maximum de tokens générés | Toujours le fixer explicitement. Un oubli = factures surprises et réponses tronquées. |
| `stop` / `stop_sequences` | Chaînes qui arrêtent la génération | Indispensable pour le formatage (ex. : stop sur `"\n\nQuestion:"` dans un chat multi-tours). |
| `seed` | Graine aléatoire | Quand le fournisseur l'expose : même prompt + même seed = même sortie (utile pour les tests). |
| `presence_penalty` / `frequency_penalty` | Pénalisent la répétition de tokens déjà utilisés | 0 à 0,5 pour éviter les boucles et la redite ; au-delà, le texte devient peu naturel. |
| `logit_bias` | Force ou bannit des tokens spécifiques | Cas avancés (ex. : interdire un token, forcer un format). |
| `response_format` | Contrainte de format (JSON, JSON schema) | **Préférer ça à « réponds en JSON » dans le prompt** quand l'API le permet : c'est garanti, pas espéré. |

## Exemple concret (API type OpenAI-compatible)

```python
from openai import OpenAI

client = OpenAI()  # lit OPENAI_API_KEY dans l'environnement

# Cas 1 : extraction factuelle → déterministe
r1 = client.chat.completions.create(
    model="gpt-5-mini",
    temperature=0,
    max_tokens=200,
    messages=[{"role": "user", "content": "Extrais la date et le montant du contrat."}],
)

# Cas 2 : rédaction → un peu de variété
r2 = client.chat.completions.create(
    model="gpt-5-mini",
    temperature=0.8,
    top_p=0.95,
    max_tokens=800,
    messages=[{"role": "user", "content": "Rédige une accroche pour un article sur les LLM."}],
)
```

## Température et hallucinations

Lien direct avec le chapitre 3 : **plus la température est haute, plus le modèle invente**. Une température élevée aplatit la distribution, donnant une chance réelle aux tokens improbables — dont les noms, dates et faits inventés.

| Tâche | Température recommandée |
|-------|--------------------------|
| Répondre à partir d'un document (RAG) | 0 – 0,2 |
| Résumé, extraction, classification | 0 – 0,3 |
| Code, traduction, reformulation fidèle | 0 – 0,3 |
| Réponse factuelle encyclopédique | 0,2 – 0,5 |
| Conversation, assistant général | 0,5 – 0,8 |
| Brainstorming, idéation | 0,8 – 1,0 |
| Création littéraire, exploration de styles | 1,0 – 1,2 |

## Pièges connus

- **Température 0 ne garantit pas la vérité** : le modèle choisit alors le token le plus probable, qui peut être faux. Le déterminisme ne résout pas les hallucinations (chapitre 3).
- **« Le modèle est créatif à température 1 »** : la créativité utile plafonne souvent vers 0,7-0,9. Au-delà, c'est surtout du bruit.
- **Les paramètres ne sont pas portables** : les plages et comportements varient selon les fournisseurs et les modèles. Toujours re-tester après un changement de modèle.
- **Oublier `max_tokens`** : le plafond par défaut est parfois très bas (ex. : 256), tronquant les réponses sans erreur visible.
- **Mesurer, ne pas deviner** : pour un cas d'usage critique, tester 2-3 réglages sur un jeu de 20-50 exemples et mesurer (voir chapitre 5 sur l'évaluation de prompts).

## Références

- [OpenAI — « API Reference : Chat completions »](https://platform.openai.com/docs/api-reference/chat) — paramètres de sampling officiels
- [Hugging Face — « Text generation strategies »](https://huggingface.co/docs/transformers/generation_strategies) — température, top-p, top-k, beam search (en anglais)
- [Holtzman et al. — « The Curious Case of Neural Text Degeneration » (arXiv, 2020)](https://arxiv.org/abs/1904.09751) — article fondateur sur top-p/top-k (en anglais)
- [Anthropic — « Prompt engineering : temperature »](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — recommandations par fournisseur (en anglais)
