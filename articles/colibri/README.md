# colibri — synthèse et mise en place d'un modèle de développement (Qwen3-Coder-30B-A3B)

> Catégorie : Guide d'installation

**Date :** 2026-10-29
**Statut :** Terminé

## Objectif

Comprendre ce qu'est [colibri](https://github.com/JustVugg/colibri) — un moteur d'inférence en C pur, sans dépendance, pour faire tourner des modèles Mixture-of-Experts (MoE) de centaines de milliards de paramètres sur du matériel ordinaire — et savoir mettre en place un **modèle dédié au développement** sur un laptop 32 Go de RAM : **Qwen3-Coder-30B-A3B** (30B paramètres, ~3B actifs par token, tool calling natif, sans raisonnement). Le guide couvre l'installation sur les deux machines cibles (laptop pro Windows + WSL, portable Linux Ubuntu), la connexion d'opencode, et conserve la procédure GLM-5.3-Flash en annexe pour référence.

## Pourquoi Qwen3-Coder-30B-A3B pour du développement

Critères retenus pour un laptop 32 Go de RAM, sans GPU dédié :

| Critère | Qwen3-Coder-30B-A3B | Pourquoi |
|---|---|---|
| Empreinte RAM | 8 Go min / 18 Go confort | Tient dans 32 Go avec l'OS et les applis ; les experts tiennent en RAM, plus de streaming disque |
| Empreinte disque | 19,4 Go (conteneur int4 gs64 pré-converti) | Tient sur un SSD de 512 Go (et 1 To) sans conversion manuelle |
| Vitesse CPU | 8,5-9,6 tok/s mesuré (machine A, tous experts en RAM) | Utilisable en interactif, contrairement aux modèles géants (~1 tok/s) |
| Spécialisation | Modèle de codage dédié (Qwen3-Coder), Apache 2.0 | Conçu pour le code et les tool calls |
| Tool calling | Oui — OpenAI `tools` + Anthropic `tool_use`, format natif `<tool_call>` XML | Fonctionne avec opencode et les CLI agentiques |
| Raisonnement | Aucun (`enable_thinking` off, le template ne le prévoit pas) | Réponses directes, pas de bloc `<think>` à attendre — mieux pour un agent de codage |
| Moteur | `c/qwen36.c` (`make qwen36`), famille `qwen3_moe` | Dans le catalogue du setup one-step (`qwen3-coder-30b`) |

**Pourquoi pas GLM-5.3-Flash (321B)** : ~25 Go de RAM minimum, ~195 Go de disque converti, ~20-44 s/token sur CPU — inutilisable en agentique sur ces machines. La procédure reste en [annexe](#annexe--mise-en-place-de-glm-53-flash).

**Pourquoi pas Qwen3.6-35B-A3B** (l'autre « petit » modèle) : aussi viable (23 Go disque, 10/20 Go RAM, ~6 tok/s), mais c'est un modèle de chat général avec raisonnement ; Qwen3-Coder est spécialisé code, un peu plus léger et plus rapide, et ne perd pas de temps en chaîne de pensée.

## Prérequis

- **Laptop pro (Windows + WSL)** ou **portable Linux Ubuntu**, 32 Go de RAM, SSD (512 Go ou 1 To)
- ~19,4 Go d'espace disque pour le modèle + quelques Go pour le cache d'experts
- `git`, Python 3.10+, un compilateur C (`gcc`/`clang` + OpenMP) ou le téléchargement du binaire précompilé
- Docker n'est **pas nécessaire** : colibri tourne en natif (WSL est du Linux natif). Docker n'est utile que pour isoler la passerelle API (voir `docker/README.md` upstream)
- Optionnel : un GPU Vulkan (iGPU comprise — le setup active Vulkan pour Qwen3-Coder sur iGPU, mesuré plus rapide sur Qwen3.6)

## Ce qu'est colibri

colibri (« tiny engine, immense model ») fait tourner de très grands modèles MoE sur la machine qu'on a déjà, **sans GPU requis**. Dans un modèle MoE, seule une petite partie des poids est active par token : les experts routés. colibri garde en RAM la partie dense (attention, embeddings, experts partagés) et **streame les experts routés depuis le disque** quand le routeur les demande, via un cache qui apprend quels experts votre usage sollicite (`.coli_usage`, mis à jour à chaque tour). Sur un modèle de la taille de Qwen3-Coder, **tous les experts tiennent en RAM** : le disque ne sert qu'au chargement initial, la génération tourne à la vitesse du CPU.

- **En C pur, zéro dépendance** : pas de BLAS, pas de framework, pas de Python à l'exécution. Un fichier `.c` par famille de modèles (`c/qwen36.c` pour Qwen3.6/Coder…) au-dessus de headers partagés (`st.h`, `quant.h`, `expert_store.h`, `tok.h`…)
- **13 moteurs** : 10 LLM (GLM-5.2/5.3, GLM-5.3-Flash, Inkling, Kimi K3, DeepSeek V4 Flash / V4.1 Flash, MiMo-V2.6 Flash/Pro, Qwen3.8-Flash-Next, Qwen3.6/Coder, OLMoE), 1 générateur d'images (Qwen-Image-2.1), 3 modèles de décision (Laya, GLiNER2.5-Decide, Clef)
- **Version 2.0.0** (2026-10-06), licence Apache 2.0
- Le Python ne sert qu'à l'installation, au launcher et à la passerelle API

## Comment ça marche

Chaque token parcourt cinq étapes dans chaque couche : **route, union, place, overlap, learn**. Le principe fondateur : **le placement ne décide que de la vitesse** — les décisions du routeur et la précision des poids sont identiques que l'expert réponde depuis la VRAM, la RAM ou le disque.

- **Trois tiers de résidence** pour les experts : VRAM (GPU), RAM (cache LRU par couche + experts épinglés appris de l'historique), disque (lu par `pread`, les trois matrices d'un expert en un seul `pread`)
- **Chevauchement** : un pool de threads lit les experts manquants pendant que les experts résidents calculent ; un thread de prefetch anticipe la couche suivante
- **Apprentissage** : `.coli_usage` enregistre la chaleur de routage mesurée ; colibri devient plus rapide plus on l'utilise
- **Spéculation** : têtes MTP (Multi-Token Prediction) sur les modèles qui en ont une (GLM-5.2, Qwen3.8-Flash-Next) ; Qwen3-Coder n'en a pas, sa vitesse vient de sa taille
- **Fidélité** : chaque moteur est vérifié en CI contre l'implémentation de référence du modèle sur une fixture tiny

## Les surfaces

| Surface | Commande | Rôle |
|---|---|---|
| Chat terminal | `coli chat --model <dir>` | Chat interactif |
| Dashboard | `coli web --model <dir>` | Chat, System One, « Brain » (atlas des experts), profilage |
| API | `coli serve --model <dir>` | OpenAI (`/v1/chat/completions`) + Anthropic (`/v1/messages`), streaming, tool calling, jusqu'à 16 conversations simultanées (`--kv-slots N`) |
| System One | `POST /v1/systemone` | Décisions à probabilité calibrée — compatible API Jev de TypeSafe |
| Planification | `coli plan --model <dir>` | Où le modèle vivra (VRAM/RAM/disque), goulot attendu |
| Diagnostic | `coli doctor --model <dir>` | Vérifie que tout est prêt |
| Réglage | `coli tune --model <dir>` | Mesure et sauvegarde les réglages les plus rapides et sûrs de la machine |
| MCP | `coli mcp` | Serveur Model Context Protocol pour piloter colibri depuis un assistant IA |
| Setup one-step | `./start-here.sh` (Linux/WSL), `START-HERE.bat` (Windows natif) | Détecte le matériel, recommande un modèle, compile ou télécharge le moteur, télécharge le modèle, démarre le dashboard |

## Mise en place de Qwen3-Coder-30B-A3B

Qwen3-Coder-30B-A3B (`Qwen/Qwen3-Coder-30B-A3B-Instruct`, Apache 2.0) : **30B paramètres, ~3B actifs par token, 48 couches, 128 experts (top-8), tool calling natif, pas de raisonnement**. Moteur `c/qwen36.c`, id catalogue `qwen3-coder-30b`, conteneur pré-converti int4 gs64 de **19,4 Go** (`Justvugg/Qwen3-Coder-30B-A3B-colibri-int4`).

### Sur le portable Linux Ubuntu (32 Go, SSD 512 Go)

Le setup one-step fait tout tout seul :

```bash
sudo apt install git python3 build-essential
git clone https://github.com/JustVugg/colibri
cd colibri
./start-here.sh --model qwen3-coder-30b
```

Ce que fait le setup :

1. **Détecte le matériel** (RAM, disque, CPU, GPU)
2. **Vérifie que le modèle tient** (dense ~3,6 Go + cache d'experts ; 18 Go confort, 8 Go minimum)
3. **Compile le moteur** `qwen36` (avec Vulkan si un GPU est détecté, CPU sinon) ou télécharge le binaire précompilé
4. **Télécharge le modèle** (19,4 Go, reprenable : `Ctrl+C`, relancer continue)
5. **Démarre le serveur** et ouvre le dashboard : `http://127.0.0.1:8000/`

Relancer ensuite : `./start-here.sh` démarre directement. `./start-here.sh --list` montre tous les modèles compatibles avec la machine. `coli status` / `coli stop` pour voir/arrêter.

Vérification rapide :

```bash
coli chat --model ~/colibri-models/<dossier-du-modele>
```

### Sur le laptop pro (Windows + WSL, 32 Go, SSD 1 To)

**Recommandé : WSL (natif, plus simple que Docker).** WSL est un Linux complet ; colibri s'y installe comme sur Ubuntu, et le SSD Windows est accessible en natif. Le dossier du modèle **doit rester sur le disque Linux de WSL** (`~/colibri-models`), jamais sous `/mnt/c` (le disque Windows vu depuis WSL est plusieurs fois plus lent).

```bash
# Dans un terminal WSL (Ubuntu)
sudo apt install git python3 build-essential
wsl$ git clone https://github.com/JustVugg/colibri
wsl$ cd colibri
wsl$ ./start-here.sh --model qwen3-coder-30b
```

Si le réseau WSL est très lent, le setup propose de télécharger via le `curl.exe` de Windows (`--via-windows auto` par défaut) — le modèle atterrit quand même dans le disque Linux.

**Alternative : Windows natif** (si tu préfères tourner sans WSL) : télécharger le ZIP du dépôt, double-cliquer sur `START-HERE.bat`, choisir `qwen3-coder-30b`. Le binaire précompilé Windows est publié (pas besoin de MinGW). Le modèle ira dans `%USERPROFILE%\colibri-models`.

**Alternative : Docker** — possible mais inutile ici : le conteneur embarque la passerelle, pas le modèle (qui fait 19,4 Go et doit rester sur un volume monté). Voir `docker/README.md` upstream. Pour un laptop 32 Go, le natif (WSL) est plus simple et plus rapide.

### Régler le cache d'experts (`--cap`)

Le cache d'experts est le seul réglage qui compte sur cette taille de modèle. Sur 32 Go, viser **tous les experts en RAM** (mesuré 8,5-9,6 tok/s) plutôt que le minimum (6,5 Go, 5,1 tok/s) :

```bash
coli serve --model ~/colibri-models/<dossier> --cap 128
```

- `--cap 128` : les 128 experts/layer en RAM (~15,2 Go résident) — **recommandé sur 32 Go**
- `--cap 32` : 6,5 Go résident, plus lent (5,1 tok/s)
- sans `--cap` : le moteur choisit (souvent 8, puis le cache LRU s'adapte)

`coli plan --model <dir>` montre le placement prévu (VRAM/RAM/disque) et le goulot attendu ; `coli tune --model <dir>` mesure et sauvegarde les réglages les plus rapides de la machine.

### Performances attendues sur ces machines

Machine de référence upstream (Ryzen 7 PRO 8700GE, 8 cœurs, DDR5, NVMe, iGPU Radeon 780M) :

| Scénario | Valeur |
|---|---|
| Decode, tous experts en RAM (`--cap 128`) | 8,5-9,6 tok/s (CPU) |
| Decode, 32 experts/layer (`--cap 32`) | 5,1 tok/s (CPU) |
| Avec Vulkan sur iGPU (Qwen3.6 mesuré) | 9,9 tok/s (tier d'experts + dense chain) |
| Résident RAM (`--cap 128`) | ~15,2 Go |
| Résident RAM (`--cap 32`) | 6,5 Go |
| Conteneur disque | 19,4 Go |

Sur un laptop 32 Go récent, attendez-vous à quelque chose entre 5 et 10 tok/s en decode selon le CPU — tout à fait utilisable pour du codage assisté, y compris en agentique.

## Utiliser colibri depuis opencode

[opencode](https://opencode.ai) est un agent de codage en terminal. colibri expose une API OpenAI-compatible (`coli serve`), donc opencode s'y connecte comme à n'importe quel fournisseur local : un provider personnalisé dans `opencode.json`, pointé sur `http://127.0.0.1:8000/v1`.

### Prérequis

- `coli serve` en cours avec le modèle chargé (le `--model-id` passé au lancement est l'identifiant exposé) :

```bash
coli serve --model ~/colibri-models/<dossier> --model-id qwen3-coder-30b-a3b-colibri --cap 128 --port 8000
```

L'identifiant par défaut est `qwen3-coder-30b-a3b-colibri` (lu dans le `config.json` du conteneur) ; `--model-id` sert à le surcharger.

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
        "qwen3-coder-30b-a3b-colibri": {
          "name": "Qwen3-Coder-30B-A3B (colibri)",
          "limit": { "context": 262144, "output": 32768 }
        }
      }
    }
  }
}
```

- `colibri` est l'identifiant du provider (libre, repris dans les références `colibri/<model>`)
- `npm: "@ai-sdk/openai-compatible"` : le paquet AI SDK pour une API `/v1/chat/completions`
- `apiKey: "local"` : valeur bidon — colibri n'exige aucune clé par défaut (la clé n'est vérifiée que si `COLI_API_KEY` est défini au lancement du serveur)
- `models` : la clé doit correspondre à l'`id` renvoyé par `GET /v1/models` ; `limit.context` n'est que l'affichage du budget côté client (Qwen3-Coder : 262 144 tokens natifs), `limit.output` le plafond de génération

Puis sélectionner le modèle dans opencode : `opencode` puis `/models`, ou directement `opencode run --model colibri/qwen3-coder-30b-a3b-colibri "ta question"`.

### Agents opencode dédiés

opencode permet de définir ses propres agents (section `agent` du `opencode.json`) avec chacun son modèle au format `provider/model-id`. Exemple : `build` sur colibri, `plan` en lecture seule, et un sous-agent `review`.

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
        "qwen3-coder-30b-a3b-colibri": {
          "name": "Qwen3-Coder-30B-A3B (colibri)",
          "limit": { "context": 262144, "output": 32768 }
        }
      }
    }
  },
  "agent": {
    "build": {
      "mode": "primary",
      "model": "colibri/qwen3-coder-30b-a3b-colibri"
    },
    "plan": {
      "mode": "primary",
      "model": "colibri/qwen3-coder-30b-a3b-colibri",
      "tools": { "write": false, "edit": false, "bash": false }
    },
    "review": {
      "description": "Relit le code sans le modifier",
      "mode": "subagent",
      "model": "colibri/qwen3-coder-30b-a3b-colibri",
      "tools": { "write": false, "edit": false, "bash": false }
    }
  }
}
```

- `build` et `plan` sont les agents primaires intégrés d'opencode, ici surchargés pour pointer sur colibri
- `review` est un sous-agent personnalisé (`mode: "subagent"`), invoquable via `@review`
- `tools: { "write": false, "edit": false, "bash": false }` restreint l'agent à la lecture — utile pour un plan ou une revue
- Le tool calling de Qwen3-Coder est supporté (OpenAI `tools` et Anthropic `tool_use`, format natif `<tool_call>` XML) — opencode peut lire, écrire et exécuter

## Annexe — mise en place de GLM-5.3-Flash (référence)

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

```bash
cd colibri/c
./glm53 --model /chemin/glm53_i4 --prompt "Bonjour" --greedy 32
# ou via le launcher :
coli chat  --model /chemin/glm53_i4 --no-think   # chat terminal
coli serve --model /chemin/glm53_i4             # API OpenAI/Anthropic + dashboard
coli web   --model /chemin/glm53_i4             # idem, ouvre le navigateur
```

`--no-think` désactive le raisonnement ; `--effort low|high|max` règle le niveau. Le tool calling est complet et le rendu est épinglé octet par octet contre le `chat_template.jinja` officiel.

### Réglages utiles (variables d'environnement)

| Variable | Défaut | Effet |
|---|---|---|
| `GLM53_BITS` | `4` | Précision des poids denses résidents : 4, 8 ou 32 bits |
| `GLM53_EXPERT_GB` | auto | Budget RAM (Go) pour le cache LRU d'experts ; unset = mémoire disponible moins 3 Go de marge |
| `GLM53_MAX_IMAGE_TOKENS` | `8000` | Tokens d'image max (28×28 px par token). 8000 = 2691 tokens pour une photo 1080p : abaisser à 256 (texte lisible) ou 64 (formes/couleurs) |
| `GLM53_PREFILL_CHUNK` | `128` | Prompt traité par tranches de N tokens (espace de travail plus petit) |
| `GLM53_MAXT` | `8192` | Capacité du cache KV en tokens |

### Performances mesurées (référence upstream)

Machine de référence : 6 cœurs physiques, 25 Go de RAM, modèle sur disque ordinaire — **cette configuration est à la limite sur un laptop 32 Go** (le cache d'experts réduit d'autant la marge).

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

Entrée image depuis `coli chat` (chemin collé dans le message), `coli web` (pièce jointe), ou `image_url` en base64 via l'API. Le prétraitement (`tools/glm53_image.py`) est épinglé contre le `Glm5NextImageProcessor` officiel. Côté serveur, un chemin local n'est lu que sous `COLI_IMAGE_ROOT` et les URL distantes sont refusées.

## Pièges connus

- **Ne pas confondre les moteurs** : `glm-5.2`/`glm-5.3` passent par le setup one-step et le moteur `colibri.c` ; GLM-5.3-Flash a son propre moteur `glm53.c` et sa propre conversion (`tools/convert_glm53.py`), hors catalogue
- **Espace disque** : Qwen3-Coder = 19,4 Go (conteneur pré-converti, rien à convertir). GLM-5.3-Flash = ~195 Go converti (328 Go source)
- **GLM-5.3-Flash sur 32 Go** : ~25 Go de RAM minimum, ~20-44 s/token — fonctionnel mais lent ; réservé à des questions ponctuelles, pas à l'agentique
- **Agent de codage (opencode, etc.) avec un grand modèle streamé** : le prefill d'un agent (prompt système + définitions d'outils, souvent 10-20k tokens) tourne à quelques tokens/seconde en CPU — comptez un long « thinking » silencieux avant le premier token. Le decode (~1 tok/s sur un gros modèle) pénalise aussi les boucles multi-tours. Sur Qwen3-Coder, tous les experts tiennent en RAM : le prefill reste le coût principal d'un prompt agentique, mais le decode (~6-10 tok/s) suit
- **`--cap` sur Qwen3-Coder** : sans valeur explicite, le moteur démarre souvent à 8 experts/layer ; `--cap 128` met tous les experts en RAM (~15,2 Go) et double la vitesse de decode (5,1 → 8,5-9,6 tok/s)
- **opencode + `apiKey`** : opencode refuse souvent de démarrer sans clé — la valeur bidon `"local"` suffit, colibri n'en exige aucune par défaut
- **opencode + tool calling** : ne fonctionne que sur les moteurs qui le supportent (Qwen3.6/Coder, GLM-5.2/5.3-Flash, Kimi K3, MiMo-V2.6, DeepSeek V4, Qwen3.8 oui ; Inkling et OLMoE non) — cf. la matrice dans la doc upstream `docs/api.md`
- **Vulkan sur glm53** : testé sur Lavapipe (Vulkan logiciel) pour la correction ; aucun GPU réel n'avait encore tourné les vrais poids upstream à la rédaction
- **WSL** : garder le dossier du modèle sur le disque Linux (`~/colibri-models`), pas sous `/mnt/c` (beaucoup plus lent). Le setup le fait par défaut ; si le réseau WSL est lent, il propose `--via-windows`
- **Docker** : inutile pour ce besoin — le natif (WSL) est plus simple et le conteneur colibri n'embarque pas le modèle de toute façon

## Références

- [Dépôt GitHub colibri](https://github.com/JustVugg/colibri) — v2.0.0, Apache 2.0
- [Site officiel](https://justvugg.github.io/colibri)
- [Page du moteur Qwen3.6 / Qwen3-Coder](https://github.com/JustVugg/colibri/blob/main/docs/qwen36.md) — architecture, conteneurs, performances
- [Qwen3-Coder-30B-A3B sur Hugging Face](https://huggingface.co/Qwen/Qwen3-Coder-30B-A3B-Instruct) — checkpoint officiel (Apache 2.0)
- [Conteneur int4 pré-converti](https://huggingface.co/Justvugg/Qwen3-Coder-30B-A3B-colibri-int4) — 19,4 Go, prêt à l'emploi
- [Page du moteur GLM-5.3-Flash](https://github.com/JustVugg/colibri/blob/main/docs/glm53-flash.md) — architecture, mémoire, tests (annexe)
- [GLM-5.3-Flash sur Hugging Face](https://huggingface.co/zai-org/GLM-5.3-Flash)
- [Quick start](https://github.com/JustVugg/colibri/blob/main/docs/quickstart.md) — installation pas à pas, toutes plateformes
- [Variables d'environnement](https://github.com/JustVugg/colibri/blob/main/docs/ENVIRONMENT.md) — inventaire complet par moteur
- [Benchmarks et protocole de mesure](https://github.com/JustVugg/colibri/blob/main/docs/benchmarks.md)
- [API, matrice de tool calling et connexion des CLI/éditeurs](https://github.com/JustVugg/colibri/blob/main/docs/api.md)
- [Guide Windows](https://github.com/JustVugg/colibri/blob/main/docs/windows.md) — build natif, START-HERE.bat
- [Documentation opencode — providers](https://opencode.ai/docs/providers/) — provider personnalisé `@ai-sdk/openai-compatible`
- [Documentation opencode — agents](https://opencode.ai/docs/agents/) — section `agent` de `opencode.json`
- [Analyse : intérêt d'un fork en Rust](fork-rust.md)
