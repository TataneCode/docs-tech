# Chapitre 10 — L'inférence : faire tourner un LLM

> Partie de : [LLM — fondamentaux et panorama](README.md)

**Date :** 2026-10-29

## Objectif

Comprendre ce que recouvre l'**inférence** — la phase où un modèle entraîné génère du texte — et quels sont les mécanismes qui la rendent utilisable en pratique : mémoire, cache KV, quantification, batching, décodage spéculatif et moteurs de serving.

## Définition

L'**inférence** (inference, « serving » en production) est la phase où les poids du modèle sont **figés** et servent à produire des sorties sur de nouvelles entrées. Contrairement à l'entraînement (chapitre 2), aucun poids n'est modifié : le modèle lit, il n'apprend pas.

```
Entrée : prompt + historique (tokens)
        ↓
Passage avant dans le réseau (chapitre 9), token par token
        ↓
Distribution de probabilités sur le vocabulaire → choix d'un token (chapitre 4)
        ↓
Le token choisi est réinjecté en entrée → boucle jusqu'au token de fin
```

La génération est **autorégressive** : chaque nouveau token dépend de tous les précédents (chapitre 1). Conséquence directe : générer K tokens coûte K passages dans le réseau, d'où l'importance des optimisations ci-dessous.

## Le goulot d'étranglement n° 1 : la mémoire

Deux problèmes de mémoire structurent toute l'inférence :

| Problème | Explication | Conséquence |
|----------|-------------|-------------|
| **Poids du modèle** | Un modèle 70B = ~140 Go en FP16 | Ne tient pas dans la mémoire d'une carte graphique classique (24-48 Go) : il faut le découper sur plusieurs GPU, ou le compresser (quantification) |
| **Cache KV** | Pour chaque token déjà traité, l'attention produit deux vecteurs (clé K, valeur V) réutilisés à chaque étape suivante | Le cache KV croît avec la longueur de la séquence et le nombre de requêtes simultanées : c'est lui qui limite le débit d'un serveur |

## Quantification : réduire la taille des poids

La **quantification** stocke les poids en précision réduite :

| Format | Bits/poids | Taille d'un 70B | Usage |
|--------|-----------|-----------------|-------|
| FP32 (entraînement) | 32 | ~280 Go | Référence, entraînement uniquement |
| FP16 / BF16 | 16 | ~140 Go | Inférence « pleine précision » |
| INT8 | 8 | ~70 Go | Compromis qualité/coût |
| INT4 (GPTQ, AWQ, GGUF Q4_K_M…) | 4 | ~35 Go | Standard du local (Ollama, llama.cpp, LM Studio) |
| 1-2 bits (ex. BitNet, LQ-LoRA) | 1-2 | ~9-18 Go | Recherche active ; perte de qualité significative |

Règles pratiques :

- Sur du matériel grand public, un modèle 7-14B en INT4 tient dans 8-16 Go de VRAM et tourne utilisablement sur GPU ; au-delà, la RAM/CPU ralentit fortement.
- La qualité chute peu entre FP16 et INT4 pour la plupart des tâches ; en dessous de 4 bits, la dégradation devient visible (résumer : testez sur votre cas d'usage, chapitre 5).
- La quantification peut être **post-entraînement** (on compresse un modèle existant) ou **aware** pendant l'entraînement (meilleure qualité à bas débit de bits).

## Le cache KV : ne pas recalculer ce qu'on connaît

À chaque nouveau token, l'attention devrait recalculer K et V pour **tous** les tokens précédents. Le **cache KV** stocke ces vecteurs au fur et à mesure : seul le nouveau token est traité, les autres sont lus en mémoire.

- Sans cache KV, générer 1 000 tokens coûterait ~500 000 opérations d'attention cumulées ; avec, 1 000.
- **Gestion paginée (PagedAttention)** : le cache KV est découpé en blocs de taille fixe alloués à la demande, comme la mémoire virtuelle d'un OS. Résultat (papier vLLM) : gaspillage quasi nul et partage de préfixes entre requêtes (ex. même prompt système pour tous les utilisateurs).
- Le cache KV se **partage** aussi entre les branches d'une même requête (beam search) et entre requêtes ayant le même préfixe (préfill partagé).

## Débit (throughput) vs latence : deux objectifs différents

| Métrique | Définition | Ce qui l'améliore |
|----------|-----------|-------------------|
| **TTFT** (time to first token) | Délai avant le premier token affiché | Prefill rapide, parallélisme, modèle petit |
| **Latence par token** (inter-token latency) | Délai entre deux tokens | Cache KV, décodage spéculatif, quantification, GPU nombreux |
| **Débit global** (tokens/s agrégés) | Tokens produits par seconde sur tout le serveur | **Batching continu** : on groupe des dizaines de requêtes actives dans le même passage GPU |

Le **batching continu** (continuous batching) insère ou retire des requêtes du batch à chaque étape de décodage, au lieu d'attendre qu'un batch entier termine : c'est ce qui multiplie par 10-100 le débit d'un serveur par rapport à des appels isolés.

## Accélérer le décodage

Puisque chaque token exige un passage dans le réseau, on cherche à en produire plusieurs par passage :

- **Décodage spéculatif** (speculative decoding) : un petit modèle « brouillon » (ou un mécanisme de n-grammes) propose K tokens d'un coup ; le grand modèle les vérifie en **un seul** passage parallèle et accepte la plus longue préfixe correct. Gain typique : ×2-3 sans changer la distribution de sortie.
- **Multi-token prediction** : le modèle prédit directement plusieurs tokens futurs (MTP), utilisé par certains modèles récents (ex. DeepSeek) aussi bien en inférence qu'en entraînement.
- **Parallélisme du modèle** : découper le réseau sur plusieurs GPU (tensor/pipeline parallel) pour que les poids tiennent en mémoire et que le calcul soit réparti.

## Où tourne l'inférence : du cloud au local

| Option | Exemples | Quand l'utiliser |
|--------|----------|------------------|
| API cloud (fournisseur) | OpenAI, Anthropic, Mistral, Google… | Zéro infra, paiement au token, données envoyées chez le fournisseur |
| Serveur dédié open source | vLLM, SGLang, TensorRT-LLM | Fort débit, données chez soi, GPU dédié |
| Sur votre machine (local) | llama.cpp, Ollama, LM Studio, MLX | Confidentialité, coût nul à l'usage, hors-ligne — voir l'article [LM Studio sur Ubuntu](../lm-studio-ubuntu/README.md) et le moteur [colibri](../colibri/README.md) |
| Navigateur / edge | WebGPU + transformers.js, MLX sur Mac | Applications légères, démo, confidentialité maximale |

Coûts indicatifs (chapitre 8 pour le détail) : une API facturée au token revient à quelques centimes par million de tokens en entrée pour les modèles récents ; un GPU cloud coûte ~2-4 €/heure ; un modèle local sur votre propre machine ne coûte que l'électricité.

## Inférence et contexte long

La fenêtre de contexte (chapitre 1) a un coût caché en inférence : le cache KV d'une séquence de 100 000 tokens occupe plusieurs Go de VRAM **par requête**. Les optimisations de contexte long visent justement à réduire cette empreinte :

- **Attention parcimonieuse** (sparse attention) : chaque token n'observe qu'une partie des précédents (ex. Mistral, Longformer).
- **Attention linéaire / états compressés** (RWKV, Mamba/SSM) : l'état du passé est résumé dans un vecteur de taille fixe, indépendant de la longueur.
- **Compression du cache KV** : moins de têtes K/V stockées, ou vecteurs de dimension réduite.

## Pièges connus

- **Confondre inférence et entraînement** : appeler une API, c'est de l'inférence ; « apprendre » au modèle pendant une conversation ne marche que dans la limite du contexte (chapitre 1). L'apprentissage persistant passe par le fine-tuning (chapitre 6).
- **Sous-estimer la VRAM du cache KV** : un modèle 7B + un contexte de 32 000 tokens tiennent facilement dans 24 Go… jusqu'à ce que 10 utilisateurs simultanés saturent le serveur. Toujours dimensionner modèle + KV cache + marge.
- **Choisir la quantification la plus agressive par défaut** : testez. Un Q4 peut suffire pour du chat, mais dégrader un raisonnement ou de l'extraction structurée (chapitre 5).
- **Négliger le TTFT** : pour une interface interactive, un premier token rapide compte plus qu'un débit élevé ; pour du batch (résumés en masse), c'est l'inverse.

## Références

- [Kwon et al. — « Efficient Memory Management for LLM Serving with PagedAttention » (vLLM, arXiv)](https://arxiv.org/abs/2309.06180) — le cache KV paginé et le batching continu
- [Leviathan et al. — « Fast Inference from Transformers via Speculative Decoding » (arXiv)](https://arxiv.org/abs/2211.17192) — le décodage spéculatif
- [Documentation vLLM](https://docs.vllm.ai/) — moteur de serving de référence (quantification, parallélisme, PagedAttention)
- [llama.cpp](https://github.com/ggml-org/llama.cpp) — inférence locale sur CPU/GPU, formats GGUF quantifiés
- [Ollama — documentation](https://github.com/ollama/ollama) — inférence locale simplifiée
- [Hugging Face — « LLM Inference » / quantifications (GGUF, GPTQ, AWQ)](https://huggingface.co/docs/transformers/llm_tutorial) — panorama des formats de quantification
- [Chapitre 1 de cette série](01-fonctionnement.md) — la boucle de génération token par token ; [chapitre 9](09-reseaux-de-neurones.md) — le réseau sous-jacent
