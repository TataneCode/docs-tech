# colibri — synthèse et mise en place de GLM-5.3-Flash

> Catégorie : Guide d'installation

**Date :** 2026-10-28
**Statut :** Terminé

## Objectif

Comprendre ce qu'est [colibri](https://github.com/JustVugg/colibri) — un moteur d'inférence en C pur, sans dépendance, pour faire tourner des modèles Mixture-of-Experts (MoE) de centaines de milliards de paramètres sur du matériel ordinaire — et savoir mettre en place son moteur dédié **GLM-5.3-Flash** (321B paramètres, vision, tool calling) sur une machine en CPU seul.

## Prérequis

- Linux (Ubuntu/Debian) pour les commandes ci-dessous ; le projet couvre aussi Windows et macOS
- ~25 Go de RAM libres minimum pour GLM-5.3-Flash (partie dense ~12 Go + cache d'experts)
- ~200 Go d'espace disque pour le modèle converti (le pic de conversion = sortie + un shard source de ~5 Go)
- `gcc` (ou clang) + OpenMP pour compiler ; Python 3.10+ pour le launcher `coli` ; `numpy` pour le convertisseur
- Optionnel : toolkit CUDA (Linux, NVIDIA), un GPU Vulkan, ou Apple Silicon (build `METAL=1`)

## Ce qu'est colibri

colibri (« tiny engine, immense model ») fait tourner de très grands modèles MoE sur la machine qu'on a déjà, **sans GPU requis**. Dans un modèle MoE, seule une petite partie des poids est active par token : les experts routés. colibri garde en RAM la partie dense (attention, embeddings, experts partagés — ~10-12 Go pour GLM-5.2/5.3) et **streame les experts routés depuis le disque** quand le routeur les demande, via un cache qui apprend quels experts votre usage sollicite (`.coli_usage`, mis à jour à chaque tour). Un GPU, quand il y en a un, héberge les experts les plus chauds et les couches denses.

- **En C pur, zéro dépendance** : pas de BLAS, pas de framework, pas de Python à l'exécution. Un fichier `.c` par famille de modèles (`c/colibri.c` pour GLM-5.2/5.3, `c/glm53.c` pour GLM-5.3-Flash…) au-dessus de headers partagés (`st.h`, `quant.h`, `expert_store.h`, `tok.h`…)
- **13 moteurs** : 10 LLM (GLM-5.2/5.3, GLM-5.3-Flash, Inkling, Kimi K3, DeepSeek V4 Flash / V4.1 Flash, MiMo-V2.6 Flash/Pro, Qwen3.8-Flash-Next, Qwen3.6/Coder, OLMoE), 1 générateur d'images (Qwen-Image-2.1), 3 modèles de décision (Laya, GLiNER2.5-Decide, Clef)
- **Version 2.0.0** (2026-10-06), licence Apache 2.0
- Le Python ne sert qu'à l'installation, au launcher, aux convertisseurs et à la passerelle API

## Comment ça marche

Chaque token parcourt cinq étapes dans chaque couche : **route, union, place, overlap, learn**. Le principe fondateur : **le placement ne décide que de la vitesse** — les décisions du routeur et la précision des poids sont identiques que l'expert réponde depuis la VRAM, la RAM ou le disque.

- **Trois tiers de résidence** pour les experts : VRAM (GPU), RAM (cache LRU par couche + experts épinglés appris de l'historique), disque (lu par `pread`, les trois matrices d'un expert en un seul `pread`)
- **Chevauchement** : un pool de threads lit les experts manquants pendant que les experts résidents calculent ; un thread de prefetch anticipe la couche suivante (le routage de GLM-5.2 est prévisible à 71,6 % une couche à l'avance)
- **Apprentissage** : `.coli_usage` enregistre la chaleur de routage mesurée ; colibri devient plus rapide plus on l'utilise
- **Multi-disque / cluster** : `COLI_MODEL_MIRROR` lit sur un second SSD (+37,5 % mesuré sur deux NVMe indépendants), `COLI_NUMA=1` entrelace sur les nœuds NUMA, un mode cluster répartit les experts sur plusieurs machines
- **Spéculation** : tête MTP (Multi-Token Prediction) en int8 sur GLM-5.2 (2,2-2,8 tokens draftés par forward) ; grammar-forced drafts pour du JSON contraint
- **Fidélité** : chaque moteur est vérifié en CI contre l'implémentation de référence du modèle sur une fixture tiny ; le cache KV de l'attention MLA de GLM-5.2 est compressé (576 flottants/token au lieu de 32 768, 57× plus petit) et persiste entre les redémarrages (`.coli_kv`)

## Les surfaces

| Surface | Commande | Rôle |
|---|---|---|
| Chat terminal | `coli chat --model <dir>` | Chat interactif ; images en entrée sur les moteurs vision |
| Dashboard | `coli web --model <dir>` | Chat, System One, « Brain » (atlas des experts), profilage, thème clair/sombre |
| API | `coli serve --model <dir>` | OpenAI (`/v1/chat/completions`) + Anthropic (`/v1/messages`), streaming, tool calling, jusqu'à 16 conversations simultanées (`--kv-slots N`) |
| System One | `POST /v1/systemone` | Décisions à probabilité calibrée (choix borné, oui/non, score) — compatible API Jev de TypeSafe |
| Planification | `coli plan --model <dir>` | Où le modèle vivra (VRAM/RAM/disque), goulot attendu |
| Diagnostic | `coli doctor --model <dir>` | Vérifie que tout est prêt |
| Réglage | `coli tune --model <dir>` | Mesure et sauvegarde les réglages les plus rapides et sûrs de la machine |
| MCP | `coli mcp` | Serveur Model Context Protocol pour installer et piloter colibri depuis un assistant IA |
| Setup one-step | `./start-here.sh` (Linux/macOS), `START-HERE.bat` (Windows) | Détecte le matériel, recommande un modèle, compile ou télécharge le moteur, télécharge le modèle, démarre le dashboard |

La passerelle HTTP n'utilise que la bibliothèque standard de Python ; les protocoles moteur↔serveur sont deux protocoles ligne/texte sur stdin/stdout (mux avec batching continu, et legacy).

## Les GPU (résumé)

- **Aucun GPU requis** : tout tourne sur CPU
- **Vulkan** (tout GPU avec un driver Vulkan 1.2) : « expert tier » (cache d'experts en VRAM, alimenté par l'historique) + « dense chain » (partie dense d'une couche en une soumission GPU). Build `make VK=1 <moteur>`, activation par `COLI_VULKAN=1`
- **CUDA** (NVIDIA, Linux) : même idée, plus les couches DeltaNet de Qwen3.6 sur la carte. Build `CUDA=1`
- **Metal** (Apple Silicon) : build `METAL=1`, activation `COLI_METAL=1`
- Le setup détecte et choisit tout seul (CUDA d'abord si le toolkit est installé, sinon Vulkan, sinon CPU)

## Mise en place de GLM-5.3-Flash

GLM-5.3-Flash (`zai-org/GLM-5.3-Flash`) : **321B paramètres, 45 couches + tête MTP, vision + tool calling**. C'est un moteur à part (`c/glm53.c`), distinct du moteur GLM-5.2/5.3 (`c/colibri.c`).

Spécificités de l'architecture :

- **KDA** (Kimi Delta Attention) sur 34 des 45 couches ; **DSA avec k-pooling** sur les 11 autres
- **mHC** (hyper-connections), partagé avec DeepSeek V4
- **SwiGLU clampé** dans le MLP texte aussi, pas seulement dans la tour vision
- **MLA absorbée** : le cache KV tient le latent 512-large (33 Ko/token, soit 1,1 Go à 32k tokens) au lieu des clés étendues (qui coûteraient 11,9 Go à 8k positions) — une identité mathématique exacte, pas une approximation

**Attention** : GLM-5.3-Flash **n'est pas dans le catalogue du setup one-step** (`c/setup_catalog.py` ne contient que `glm-5.2` et `glm-5.3` pour cette famille). L'installation se fait à la main, en trois étapes.

### Étape 1 — Compiler le moteur

```bash
git clone https://github.com/JustVugg/colibri.git
cd colibri/c
make glm53              # CPU seul
# ou avec Vulkan : make VK=1 glm53
# ou avec CUDA (NVIDIA, Linux) : make CUDA=1 glm53
```

### Étape 2 — Télécharger et convertir le modèle

Le moteur lit un **conteneur converti**, pas le snapshot Hugging Face. Une seule passe télécharge et convertit, **shard par shard** (jamais les 328 Go du dépôt d'un coup) :

```bash
cd colibri/c
python3 tools/convert_glm53.py --outdir /chemin/glm53_i4 --min-free-gb 30
```

- Les experts routés (97 % des octets) passent de FP8 e4m3 à **int4 group-scaled gs64** (même conteneur que GLM-5.2)
- Tout le reste (9,7B paramètres, ~3 %, soit ~18 Go) reste en **BF16** : la précision des denses reste un choix au chargement (`GLM53_BITS`), sans rien retélécharger
- Les 92 types de tenseurs sont classés explicitement ; un nom non reconnu **arrête** la conversion (pas de checkpoint silencieusement incomplet)
- Reprise sur interruption : `Ctrl+C`, puis relancer continue où ça s'est arrêté
- Mesuré upstream : 62 shards, **194,7 Go en sortie, ~25 h** ; pic disque = sortie + un shard source de 5 Go

### Étape 3 — Lancer

Directement avec le moteur :

```bash
cd colibri/c
./glm53 --model /chemin/glm53_i4 --prompt "Bonjour" --greedy 32
```

Ou via le launcher (c'est là que vivent le template de chat et les images) :

```bash
coli chat  --model /chemin/glm53_i4 --no-think   # chat terminal
coli serve --model /chemin/glm53_i4             # API OpenAI/Anthropic + dashboard
coli web   --model /chemin/glm53_i4             # idem, ouvre le navigateur
```

`--no-think` désactive le raisonnement (le prompt ouvre `<think>`, le modèle le ferme) ; `--effort low|high|max` règle le niveau. Le tool calling est complet et le rendu est épinglé octet par octet contre le `chat_template.jinja` officiel.

### Réglages utiles (variables d'environnement)

| Variable | Défaut | Effet |
|---|---|---|
| `GLM53_BITS` | `4` | Précision des poids denses résidents : 4, 8 ou 32 bits |
| `GLM53_EXPERT_GB` | auto | Budget RAM (Go) pour le cache LRU d'experts ; unset = mémoire disponible moins 3 Go de marge |
| `GLM53_MAX_IMAGE_TOKENS` | `8000` | Tokens d'image max (28×28 px par token). 8000 = 2691 tokens pour une photo 1080p : abaisser à 256 (texte lisible) ou 64 (formes/couleurs) |
| `GLM53_PREFILL_CHUNK` | `128` | Prompt traité par tranches de N tokens (espace de travail plus petit) |
| `GLM53_MAXT` | `8192` | Capacité du cache KV en tokens |

### Performances mesurées (référence upstream)

Machine de référence : 6 cœurs physiques, 25 Go de RAM, modèle sur disque ordinaire.

| Métrique | Valeur |
|---|---|
| Poids résidents (`GLM53_BITS=4`) | ~12 Go |
| Cache KV | 33 Ko/token (1,1 Go à 32k) |
| Espace de prefill | ~60 Mo, plat quel que soit le contexte |
| Decode | ~44 s/token froid, ~20 s/token chaud |
| Octets lus/token | 4,8 Go (42 couches × 8 experts × 14,2 Mo) |
| Plancher disque | ~24 s/token (le disque plafonne à ~200 Mo/s en QD16) |

**Le disque est le mur** : aucun CPU ni GPU ne va plus vite que le temps de lecture des octets. C'est pourquoi le chemin Vulkan sert ici à *héberger* des experts en VRAM, pas à accélérer le calcul.

### Vision

Entrée image depuis `coli chat` (chemin collé dans le message), `coli web` (pièce jointe), ou `image_url` en base64 via l'API. Le prétraitement (`tools/glm53_image.py`) est épinglé contre le `Glm5NextImageProcessor` officiel. Côté serveur, un chemin local n'est lu que sous `COLI_IMAGE_ROOT` et les URL distantes sont refusées (pas de lecture réseau au gré du client).

## Utiliser colibri depuis opencode

[opencode](https://opencode.ai) est un agent de codage en terminal. colibri expose une API OpenAI-compatible (`coli serve`), donc opencode s'y connecte comme à n'importe quel fournisseur local : un provider personnalisé dans `opencode.json`, pointé sur `http://127.0.0.1:8000/v1`.

### Prérequis

- `coli serve` en cours avec le modèle chargé (le `--model-id` passé au lancement est l'identifiant exposé) :

```bash
coli serve --model /chemin/glm53_i4 --model-id glm-5.3-flash-colibri --port 8000
```

- Vérifier que l'endpoint répond (aucune clé requise par défaut) :

```bash
curl http://127.0.0.1:8000/v1/models
```

### Configuration

Dans `~/.config/opencode/opencode.json` (global) ou `opencode.json` à la racine du projet :

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "colibri": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "colibri (local)",
      "options": {
        "baseURL": "http://127.0.0.1:8000/v1",
        "apiKey": "local"
      },
      "models": {
        "glm-5.3-flash-colibri": {
          "name": "GLM-5.3-Flash (colibri)",
          "limit": { "context": 131072, "output": 8192 }
        }
      }
    }
  }
}
```

- `colibri` est l'identifiant du provider (libre, repris dans les références `colibri/<model>`)
- `npm: "@ai-sdk/openai-compatible"` : le paquet AI SDK pour une API `/v1/chat/completions`
- `apiKey: "local"` : valeur bidon — colibri n'exige aucune clé par défaut (la clé n'est vérifiée que si `COLI_API_KEY` est défini au lancement du serveur)
- `models` : la clé doit correspondre à l'`id` renvoyé par `GET /v1/models` ; `limit.context` n'est que l'affichage du budget côté client — calez-le sur ce que votre configuration KV permet réellement (ex. `GLM53_MAXT=8192` → 8192)

Puis sélectionner le modèle dans opencode : `opencode` puis `/models`, ou directement `opencode run --model colibri/glm-5.3-flash-colibri "ta question"`.

### Agents opencode dédiés

opencode permet de définir ses propres agents (section `agent` du `opencode.json`) avec chacun son modèle au format `provider/model-id`. Exemple : un agent `plan` sur le même serveur colibri, et un sous-agent `review` en lecture seule.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "colibri": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "colibri (local)",
      "options": {
        "baseURL": "http://127.0.0.1:8000/v1",
        "apiKey": "local"
      },
      "models": {
        "glm-5.3-flash-colibri": {
          "name": "GLM-5.3-Flash (colibri)",
          "limit": { "context": 131072, "output": 8192 }
        }
      }
    }
  },
  "agent": {
    "build": {
      "mode": "primary",
      "model": "colibri/glm-5.3-flash-colibri"
    },
    "plan": {
      "mode": "primary",
      "model": "colibri/glm-5.3-flash-colibri",
      "tools": { "write": false, "edit": false, "bash": false }
    },
    "review": {
      "description": "Relit le code sans le modifier",
      "mode": "subagent",
      "model": "colibri/glm-5.3-flash-colibri",
      "tools": { "write": false, "edit": false, "bash": false }
    }
  }
}
```

- `build` et `plan` sont les agents primaires intégrés d'opencode, ici surchargés pour pointer sur colibri
- `review` est un sous-agent personnalisé (`mode: "subagent"`), invoquable via `@review`
- `tools: { "write": false, "edit": false, "bash": false }` restreint l'agent à la lecture — utile pour un plan ou une revue

### Adapter à un modèle plus petit

Si GLM-5.3-Flash est trop lent pour un usage agentique (voir les avertissements ci-dessous), le même bloc fonctionne avec n'importe quel modèle servi par colibri — par exemple Qwen3.6-35B-A3B, bien plus rapide sur CPU (~6 tok/s) et suffisant pour du codage assisté :

```json
{
  "provider": {
    "colibri": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "colibri (local)",
      "options": {
        "baseURL": "http://127.0.0.1:8000/v1",
        "apiKey": "local"
      },
      "models": {
        "qwen36-colibri": {
          "name": "Qwen3.6-35B-A3B (colibri)",
          "limit": { "context": 262144, "output": 8192 }
        }
      }
    }
  }
}
```

et les agents référencent `colibri/qwen36-colibri`.

## Pièges connus

- **Ne pas confondre les moteurs** : `glm-5.2`/`glm-5.3` passent par le setup one-step et le moteur `colibri.c` ; GLM-5.3-Flash a son propre moteur `glm53.c` et sa propre conversion (`tools/convert_glm53.py`), hors catalogue
- **Espace disque** : prévoir ~195 Go pour le modèle converti, pas seulement les 328 Go du dépôt source ; le convertisseur vérifie l'espace libre (`--min-free-gb`, 30 Go constatés upstream)
- **Images trop grandes** : sans `GLM53_MAX_IMAGE_TOKENS`, une photo 1080p = 2691 tokens de prefill, interminable sur un moteur qui streame depuis le disque
- **Attentes** : ~20-44 s/token sur CPU avec disque ordinaire — c'est le prix du « 321B sur un laptop », pas un bug
- **Agent de codage (opencode, etc.) avec un grand modèle streamé** : le prefill d'un agent (prompt système + définitions d'outils, souvent 10-20k tokens) tourne à quelques tokens/seconde en CPU — comptez un long « thinking » silencieux avant le premier token. Le decode (~1 tok/s sur un gros modèle) pénalise aussi les boucles multi-tours. Les requêtes chirurgicales à contexte court passent ; les sessions agentiques itératives contre un modèle 744B streamé depuis le disque ne ressemblent pas à une API hébergée. Réduire le preamble système et le catalogue d'outils du client aide. Sur un modèle plus petit et résident en RAM (Qwen3.6, OLMoE), l'expérience est nettement plus fluide
- **opencode + `apiKey`** : opencode refuse souvent de démarrer sans clé — la valeur bidon `"local"` suffit, colibri n'en exige aucune par défaut
- **opencode + tool calling** : ne fonctionne que sur les moteurs qui le supportent (GLM-5.3-Flash oui ; Inkling et OLMoE non) — cf. la matrice dans la doc upstream `docs/api.md`
- **Vulkan sur glm53** : testé sur Lavapipe (Vulkan logiciel) pour la correction ; aucun GPU réel n'avait encore tourné les vrais poids upstream à la rédaction
- **WSL** : garder le dossier du modèle sur le disque Linux (`~/colibri-models`), pas sous `/mnt/c` (beaucoup plus lent)

## Références

- [Dépôt GitHub colibri](https://github.com/JustVugg/colibri) — v2.0.0, Apache 2.0
- [Site officiel](https://justvugg.github.io/colibri)
- [Page du moteur GLM-5.3-Flash](https://github.com/JustVugg/colibri/blob/main/docs/glm53-flash.md) — architecture, mémoire, tests
- [GLM-5.3-Flash sur Hugging Face](https://huggingface.co/zai-org/GLM-5.3-Flash)
- [Quick start](https://github.com/JustVugg/colibri/blob/main/docs/quickstart.md) — installation pas à pas, toutes plateformes
- [Variables d'environnement](https://github.com/JustVugg/colibri/blob/main/docs/ENVIRONMENT.md) — inventaire complet par moteur
- [Benchmarks et protocole de mesure](https://github.com/JustVugg/colibri/blob/main/docs/benchmarks.md)
- [API et connexion des CLI/éditeurs (aider, crush, pi…)](https://github.com/JustVugg/colibri/blob/main/docs/api.md#connect-a-coding-cli-or-editor)
- [Documentation opencode — providers](https://opencode.ai/docs/providers/) — provider personnalisé `@ai-sdk/openai-compatible`
- [Documentation opencode — agents](https://opencode.ai/docs/agents/) — section `agent` de `opencode.json`
- [Analyse : intérêt d'un fork en Rust](fork-rust.md)
