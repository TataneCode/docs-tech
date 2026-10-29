# Chapitre 8 — Bilan des modèles et de leur utilisation

> Partie de : [LLM — fondamentaux et panorama](README.md)

**Date :** 2026-10-29
**Dernière mise à jour du marché : octobre 2026**

## Objectif

Disposer d'un panorama à jour des principaux modèles de LLM (fermés et open weights), comprendre leurs positionnements, et savoir lequel choisir selon le critère qui compte pour vous : qualité, coût, latence, confidentialité, souveraineté.

> **Ce chapitre date vite.** Le marché des LLM évolue toutes les quelques semaines. Les noms et ordres de grandeur ci-dessous reflètent octobre 2026 ; les *critères de choix* (dernière section) restent valables.

## Les deux grandes familles

| | Modèles fermés (API) | Modèles open weights |
|---|---|---|
| **Exemples** | GPT (OpenAI), Claude (Anthropic), Gemini (Google), Grok (xAI), Mistral API | Llama (Meta), Qwen (Alibaba), DeepSeek, GLM (Z.ai), Kimi (Moonshot), Mistral open, Gemma (Google), MiMo (Xiaomi), gpt-oss (OpenAI) |
| **Accès** | API payante au token | Poids téléchargeables (Hugging Face), déployés sur votre infra ou chez un hébergeur |
| **Avantages** | Qualité frontier, zéro infra, mises à jour transparentes | Confidentialité (données chez vous), coût prévisible à l'échelle, personnalisable (fine-tuning), pas de dépendance fournisseur |
| **Inconvénients** | Coût au token, données envoyées chez le fournisseur, dépendance | Infra à opérer (GPU), compétence requise, qualité souvent un cran en dessous du frontier |
| **Licences** | Contrats commerciaux | Variable : MIT (DeepSeek, GLM), Apache 2.0 (Mistral Large, Qwen3, gpt-oss), licences communautaires restrictives (Llama : seuil de 700 M d'utilisateurs mensuels), licences personnalisées (Kimi, certaines versions Qwen récentes) |

**Attention** : « open weights » ≠ « open source ». Vérifier la licence avant un usage commercial (chapitre 2, phase 1 ; les licences sont listées sur les cards Hugging Face de chaque modèle).

## Panorama octobre 2026

### Modèles fermés (frontier)

| Modèle | Éditeur | Positionnement | Quand le choisir |
|--------|---------|---------------|------------------|
| **Claude Opus 5.5** | Anthropic | Référence sur l'agentique et le code complexe ; très bon rapport qualité/prix dans sa gamme | Tâches complexes, agents, code, quand la qualité prime sur le coût |
| **Claude Sonnet 5.5** | Anthropic | Excellent rapport qualité/prix ; leader sur plusieurs benchmarks de code agentique | Usage courant intensif, bon compromis qualité/coût |
| **Claude Haiku 5.5** | Anthropic | Le plus rapide et le moins cher de la famille | Haut volume, tâches simples, sous-agents |
| **GPT-6 Astra / GPT-6.x Sol** | OpenAI | GPT-6 Astra : très fort en raisonnement (maths, sciences) ; GPT-6.1 Sol : meilleur rapport résultat/prix du marché | Raisonnement pointu (Astra) ; usage général économique (Sol) |
| **Gemini 3.x / 4.x (Pro, Flash)** | Google | Multimodal natif (texte, image, audio, vidéo), très grandes fenêtres de contexte, intégration écosystème Google | Documents longs, multimodal, écosystème Google |
| **Grok 4.x** | xAI | Bon en temps réel (intégration X), compétitif en raisonnement | Veille temps réel, intégration réseaux sociaux |

### Modèles open weights (leaders)

| Modèle | Éditeur | Licence | Positionnement | Quand le choisir |
|--------|---------|---------|---------------|------------------|
| **MiMo-V2.6-Pro** | Xiaomi | Ouverte | Meilleur open weights sur l'index composite Artificial Analysis (oct. 2026) | Qualité max en local / souveraineté |
| **GLM-5.3 (+ Flash)** | Z.ai | MIT | Très fort en code agentique et tool use ; GLM-5.3-Flash très économique | Code, agents, hébergement local économique (voir article [colibri](../colibri/README.md)) |
| **Kimi K3 / K2.x** | Moonshot AI | Licence propre (à vérifier) | Excellent en code agentique et contexte très long (jusqu'à 1 M de tokens) | Tâches agentiques longues, dépôts de code entiers |
| **DeepSeek V4 (Pro / Flash)** | DeepSeek | MIT | Meilleur rapport qualité/prix open ; très bon en raisonnement et code | Usage économique à grande échelle, fine-tuning |
| **Qwen3.x / 3.8 (incl. Coder, VL)** | Alibaba | Apache 2.0 (versions ouvertes) | Large gamme de tailles, excellent multilingue (dont français), variantes code et vision | Souveraineté, multilingue, petits modèles embarqués |
| **Mistral Large 3 / Medium 3.5** | Mistral AI | Apache 2.0 | Référence européenne, très bon en français | Souveraineté européenne, langue française |
| **Llama 4 (Scout, Maverick)** | Meta | Licence communautaire Llama | Écosystème le plus large (outils, quantifications, hébergeurs) | Base solide pour fine-tuning, communauté |
| **Gemma 4** | Google | Licence Gemma (permissive) | Petits modèles efficaces, edge/on-device | embarqué, mobile, coût minimal |
| **gpt-oss** | OpenAI | Apache 2.0 | Poids ouverts d'OpenAI, raisonnement intégré | Alternative souveraine avec une philosophie familière |

**État du marché en octobre 2026** : l'écart entre le meilleur modèle fermé et le meilleur open weights s'est fortement réduit — sur le code et le raisonnement, les meilleurs modèles ouverts sont à quelques points des frontier, pour un coût par tâche souvent 5 à 15 fois inférieur. Les laboratoires chinois (DeepSeek, Alibaba, Z.ai, Moonshot, Xiaomi) mènent la frontier ouverte ; Mistral reste la référence européenne ; Meta a basculé son modèle phare (Muse Spark) en fermé, recentrant Llama sur le milieu de gamme.

## Matrice de décision

| Votre priorité | Modèles fermé | Open weights / local |
|----------------|---------------|----------------------|
| **Qualité max, sans contrainte de coût** | Claude Opus 5.5, GPT-6 Astra, Gemini Pro | MiMo-V2.6-Pro, GLM-5.3, Kimi K3 (si infra suffisante) |
| **Meilleur rapport qualité/prix** | Claude Sonnet 5.5, GPT-6.1 Sol | DeepSeek V4 (Pro/Flash), GLM-5.3-Flash |
| **Coût minimal / haut volume** | Haiku 5.5, Gemini Flash, GPT mini | Qwen3 petits modèles, Gemma 4, DeepSeek V4 Flash (local ou API) |
| **Confidentialité / données sensibles** | Contrat entreprise (DPA, hébergement UE) | **Local obligatoire** : Qwen, Mistral, DeepSeek, GLM sur votre infra |
| **Souveraineté européenne** | Mistral API (hébergement UE) | **Mistral open weights** en local |
| **Français de qualité** | Claude, Mistral API, Gemini | **Mistral** (nativement entraîné sur le français) |
| **Code / agentique** | Claude (Opus/Sonnet), GPT-6 Astra | GLM-5.3, Kimi K3, DeepSeek V4, Qwen3-Coder |
| **Contexte très long (documents, code)** | Gemini (1 M+), Llama 4 Scout (10 M) | Kimi K3 (1 M), MiniMax, Nemotron (1 M) |
| **Multimodal (image, audio, vidéo)** | Gemini, GPT, Claude (vision) | Qwen-VL, GLM (vision) |
| **Fine-tuning facilité** | API fine-tuning fournisseurs | Tout open weights ; écosystème le plus riche autour de Llama, Qwen, Mistral |

## Choisir selon le critère technique

### Par taille de modèle (open weights)

| Taille | Usage | Exemples | Matériel indicatif |
|--------|-------|----------|-------------------|
| 1-4B | Edge, embarqué, tâches simples, clasificación | Gemma 4, Qwen3 4B, SmolLM | CPU / téléphone / Raspberry Pi 5 |
| 7-14B | Usage local courant, bon compromis | Qwen3 8B/14B, Mistral 7B/12B, Llama 3.x 8B | 1 GPU 8-16 Go (quantifié) ou 16-32 Go de RAM (CPU) |
| 30-70B | Qualité locale sérieuse | Llama 3.3 70B, Qwen3 32B/72B, Mistral Large (quantifié) | 1-2 GPU 24 Go, ou 48-64 Go de RAM (CPU lent) |
| 100B+ (MoE) | Qualité proche du frontier en local | DeepSeek V4, GLM-5.3, Kimi K3, Qwen3 MoE | Cluster GPU, ou moteur spécialisé (voir [colibri](../colibri/README.md) : MoE géants en CPU/RAM/disque) |

### Par critère de coût

- **API** : comparer le prix au million de tokens (entrée/sortie). Attention aux différences entrée vs sortie (souvent x3 à x5).
- **Local** : coût = achat/location du matériel, amorti. À partir de quelques milliards de tokens/mois, le local devient moins cher que l'API.
- **Astuce** : router les requêtes simples vers un petit modèle (Haiku, Flash, Qwen3 4B) et les complexes vers un gros — le « model routing » divise souvent la facture par 3 à 10.

### Par latence

- Les modèles **Flash/Light/Mini** répondent en quelques centaines de ms (utile pour l'UX temps réel).
- Les modèles de **raisonnement** (« thinking », « Max ») réfléchissent des dizaines de secondes : excellents pour la qualité, mauvais pour l'interactivité.
- Le **local** a une latence fixe (pas de file d'attente) mais dépend du matériel.

## Sources de veille pour rester à jour

- [Artificial Analysis — LLM Leaderboard](https://artificialanalysis.ai/leaderboards/models) — intelligence, vitesse, prix, mis à jour en continu
- [LMSys Chatbot Arena](https://chat.lmsys.org/) — classements par préférences humaines (Elo)
- [Hugging Face — Open LLM Leaderboard](https://huggingface.co/spaces/HuggingFaceH4/open_llm_leaderboard) — benchmarks des modèles open weights
- [Cards Hugging Face des modèles](https://huggingface.co/models) — poids, licences, tailles, quantifications
- [Sites des éditeurs](https://www.anthropic.com/claude) : [Anthropic](https://www.anthropic.com/claude), [OpenAI](https://openai.com/), [Google AI](https://ai.google.dev/), [Mistral](https://mistral.ai/), [DeepSeek](https://www.deepseek.com/), [Qwen](https://qwen.ai/), [Z.ai (GLM)](https://z.ai/), [Moonshot (Kimi)](https://www.moonshot.ai/), [Xiaomi MiMo](https://www.mimo.ai/)

## Pièges connus

- **Benchmark ≠ usage réel** : un modèle premier sur MMLU n'est pas forcément le meilleur pour VOTRE tâche. Toujours tester sur 20-50 cas réels avant de choisir.
- **Les benchmarks sont « saturés »** : les écarts entre modèles se jouent maintenant sur des tâches spécialisées (code agentique, tool use, instruction complexe) plus que sur les généralistes.
- **Changer de modèle coûte** : chaque migration invalide une partie du prompt engineering (chapitre 5). Prévoir un jeu de test réutilisable.
- **Le prix affiché n'est pas le prix payé** : penser aux tokens de contexte (system prompt, RAG), aux retries, au raisonnement étendu.
- **Vérifier la licence avant de déployer** : « gratuit » ou « open weights » ne veut pas dire « libre de droits » (Llama, Kimi, certaines versions Qwen ont des restrictions).

## Références

- [Artificial Analysis — LLM Leaderboard](https://artificialanalysis.ai/leaderboards/models) — classement indépendant de référence
- [LMSys Chatbot Arena](https://chat.lmsys.org/) — préférences utilisateurs (Elo)
- [Hugging Face — Open LLM Leaderboard](https://huggingface.co/spaces/HuggingFaceH4/open_llm_leaderboard)
- [Hugging Face — modèles](https://huggingface.co/models) — poids, licences, quantifications
- [LM Studio sur Ubuntu (article local)](../lm-studio-ubuntu/README.md) — faire tourner des modèles open weights en local
- [colibri (article local)](../colibri/README.md) — moteur d'inférence pour grands modèles MoE sur matériel ordinaire
- [FreeLLMAPI (article local)](../freellmapi/README.md) — agréger les free tiers de fournisseurs LLM derrière une API unifiée
