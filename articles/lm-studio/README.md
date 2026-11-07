# LM Studio — héberger des LLM en local sur Ubuntu (ThinkPad T14, 32 Go en CPU)

> Catégorie : Guide d'installation

**Date :** 2026-11-04
**Statut :** Terminé

## Objectif

Installer LM Studio sur Ubuntu (base Debian) et y héberger des modèles LLM locaux (GGUF) sur un ThinkPad T14 (Ryzen 5 PRO 5450U, 32 Go de RAM, sans GPU dédié), en natif et/ou Docker : choisir ses modèles (Qwen3 8B, Gemma 4 12B, Gemma 2 9B abliterated), les télécharger même quand Hugging Face est bloqué par le pare-feu d'entreprise (catalogue LM Studio, miroirs, transit par un drive), connaître les modèles abliterated disponibles, exposer le serveur API local, connecter OpenCode pour un usage agentique, et surveiller la RAM.

## Prérequis

- Ubuntu (base Debian), compte utilisateur avec `sudo` pour l'installation native
- ~10 Go d'espace disque libre par modèle téléchargé (plus si plusieurs quantifications)
- Machine cible de référence : ThinkPad T14 Gen 2 AMD, Ryzen 5 PRO 5450U (6c/12t, Zen 3), iGPU Vega 7, 32 Go DDR4-3200
- Aucun GPU dédié : l'inférence se fait sur CPU, le modèle vit en RAM système
- Pour la section OpenCode : OpenCode installé (voir [opencode.ai](https://opencode.ai))
- Pour le transit par drive : `rclone` configuré sur un compte Drive, ou l'interface web de Drive

## Ce qu'est LM Studio

LM Studio est une application de bureau basée sur llama.cpp qui permet de chercher, télécharger et exécuter des LLM localement au format GGUF, sans dépendre d'un service cloud. Elle fournit :

- une interface graphique de chat avec réglages de sampling,
- la recherche et le téléchargement de modèles directement dans l'app (catalogue LM Studio + dépôts Hugging Face),
- un serveur API local compatible OpenAI (`http://localhost:1234/v1`),
- une CLI `lms` pour télécharger, lister, charger et servir des modèles sans ouvrir la GUI.

Le daemon headless `llmster` (LM Studio sans GUI) permet aussi un usage serveur, en natif ou en conteneur — voir la section Déploiement Docker.

## Lire sa configuration matérielle

Avant tout, vérifier la machine :

```bash
lscpu | grep -E "Model name|Thread|Core"
free -h
```

Pourquoi la config change tout :

- **32 Go de RAM sans GPU dédié** : le modèle est chargé en RAM système, partagée avec l'OS et vos applications. Le fichier GGUF ne doit pas dépasser ~60 % de la RAM libre pour rester confortable.
- **iGPU Vega 7** : elle partage la même RAM. L'accélération iGPU n'apporte quasiment rien face au CPU sur cette génération ; ne pas compter dessus.
- **DDR4-3200 double canal** : la bande passante mémoire est le facteur limitant de la vitesse de génération sur CPU. Vérifier que les deux barrettes sont présentes (`sudo dmidecode -t memory | grep -E "Size|Speed"`).

## Installation sur Ubuntu

### Option 1 : AppImage (recommandée, sans sudo)

```bash
# Télécharger depuis https://lmstudio.ai (bouton Linux, fichier .AppImage)
cd ~/Téléchargements
chmod +x LM-Studio-*.AppImage
./LM-Studio-*.AppImage
```

Si l'AppImage ne se lance pas, installer FUSE (fréquent sur Ubuntu récent) :

```bash
sudo apt update && sudo apt install libfuse2
```

### Option 2 : paquet .deb officiel

```bash
sudo dpkg -i LM-Studio-*.deb
```

L'application apparaît alors dans le lanceur d'applications.

### Premier lancement

Au premier démarrage, LM Studio détecte le matériel (CPU, RAM) et propose des réglages par défaut. Sur cette machine, il privilégiera le CPU ; c'est normal.

## Choisir ses modèles sur 32 Go en CPU

Règle empirique : **taille du fichier GGUF ≈ RAM nécessaire × 1,2** (marge pour le contexte KV et l'OS). Surveiller `free -h` pendant l'inférence.

| Modèle | Quantif. | Taille GGUF | Verdict sur T14 |
|--------|----------|-------------|-----------------|
| Qwen2.5-1.5B-Instruct | Q4_K_M | ~1 Go | Très réactif, usage quotidien léger |
| Qwen2.5-3B-Instruct | Q4_K_M | ~2 Go | Réactif, bon compromis qualité/vitesse |
| Qwen2.5-7B-Instruct | Q4_K_M | ~4,7 Go | Sweet spot historique de cette machine |
| Qwen2.5-14B-Instruct | Q4_K_M | ~9 Go | Possible à 32 Go, nettement plus lent |
| Qwen2.5-32B et plus | — | 20 Go+ | À éviter : swap, expérience dégradée |

### Fiches des modèles retenus

#### Qwen3 8B (`unsloth/Qwen3-8B-GGUF`, Q4_K_M ≈ 5 Go)

Successeur de Qwen2.5 en dense 8B : raisonnement hybride (mode thinking activable/désactivable), **tool calling natif** (indispensable pour un agent), contexte natif 32K (jusqu'à 128K avec les variantes YaRN). Sur cette machine en Q4_K_M, compter ~10-20 tok/s. C'est le daily driver recommandé ici : chat, résumé, et agentique léger via OpenCode.

#### Gemma 4 12B (`lmstudio-community/gemma-4-12B-it-QAT-GGUF` ou `unsloth/gemma-4-12b-it-GGUF`, Q4 ≈ 7 Go)

Famille Gemma 4 de Google (avril 2026, licence Apache 2.0) : E2B, E4B, **12B**, 26B-A4B (MoE) et 31B dense — il n'existe pas de variante 13B, le 12B est la taille intermédiaire. Le 12B est multimodal unifié (texte + image + audio), supporte le **tool use** et le reasoning, avec un contexte natif de 256K. Le checkpoint officiel QAT Q4_0 fait ~7 Go et est pensé pour llama.cpp/LM Studio. Sur 32 Go en CPU, c'est le meilleur rapport capacité/taille de la famille ; en pratique, limiter le contexte servi à 8-32K (le cache KV de 256K ne tiendrait pas en RAM).

#### Gemma 2 9B abliterated (`bartowski/gemma-2-9b-it-abliterated-GGUF`, Q4_K_M ≈ 5,4 Go)

Version « uncensored » de `google/gemma-2-9b-it` (ablation des vecteurs de refus, voir section suivante). ~5,4 Go en Q4_K_M, donc confortable sur cette machine. **Contexte natif limité à 8K** : rester sur des tâches courtes. À réserver à un usage personnel où le modèle de base refuse à tort (voir les avertissements de la section abliterated).

Conseils généraux :

- Privilégier les quantifications **Q4_K_M** (ou les QAT Q4_0 officiels de Google pour Gemma 4) : bon compromis qualité/taille. Éviter Q2/Q3 (cohérence dégradée) sauf contrainte forte.
- Un seul modèle chargé à la fois en mémoire.
- Sur CPU, viser 10-20 tok/s pour un 8B ; si c'est plus lent, vérifier la RAM libre et le double canal.

## Premier modèle et chat

1. Onglet de recherche (loupe), taper `qwen3 8b`.
2. Choisir un dépôt GGUF (éditeur officiel, `unsloth`, `lmstudio-community`...), puis la quantification `Q4_K_M`.
3. Télécharger (~5 Go), puis charger le modèle dans l'onglet chat.
4. Ajuster la longueur de contexte : rester modeste (4096-8192 tokens) pour ne pas manger la RAM inutilement.
5. Tester un prompt, surveiller la vitesse de génération affichée.

En CLI, le même flux existe : `lms get qwen3-8b --gguf`, `lms ls`, `lms load <clé-du-modèle>`.

## Modèles abliterated disponibles

**Abliteration** = modification des poids (orthogonalisation des directions de « refus » dans les activations) pour qu'un modèle instruction ne refuse plus certaines catégories de requêtes. Ce n'est pas un réentraînement : la taille du fichier est identique au modèle de base, la qualité peut légèrement baisser (effet « lobotomisé » signalé par la communauté sur certains dépôts).

Où trouver des modèles abliterated :

- Recherche Hugging Face avec le filtre `abliterated` : [huggingface.co/models?other=abliterated](https://huggingface.co/models?other=abliterated)
- Organisation de référence **huihui-ai** : centaines de dépôts (safetensors + GGUF) couvrant Qwen 3/3.5/3.6/3.8, Gemma 3n/4, gpt-oss, Llama 3.3/4, GLM, DeepSeek-R1 — [huggingface.co/huihui-ai](https://huggingface.co/huihui-ai)
- Conversions GGUF de qualité : `bartowski`, `mradermacher`, `QuantFactory` ; méthode originale : `IlyaGusev/gemma-2-9b-it-abliterated`
- Méthode « Heretic/MPOA » : `grimjim` (ex. `Stabhappy/gemma-4-31B-it-heretic-GGUF`, ~17 Go en Q4)
- Sur Ollama (si installé) : namespace `huihui_ai` (`ollama pull huihui_ai/qwen3.6-abliterated`)

Exemples qui tiennent sur 32 Go en CPU :

| Modèle abliterated | Dépôt GGUF | Q4_K_M | Usage |
|--------------------|------------|--------|-------|
| Gemma 2 9B | `bartowski/gemma-2-9b-it-abliterated-GGUF` | ~5,4 Go | Chat général sans refus, contexte 8K |
| Qwen3.5 9B | `huihui-ai/Huihui-Qwen3.5-9B-abliterated(-GGUF)` | ~5-6 Go | Chat + tool calling allégé |
| Qwen3 14B | `richardyoung/qwen3-14b-abliterated` (Ollama) / huihui-ai | ~9 Go | Plus capable, plus lent sur CPU |
| Gemma 4 31B Heretic | `Stabhappy/gemma-4-31B-it-heretic-GGUF` | ~17 Go | Le plus fort de la famille, très lent sur CPU (quelques tok/s) |

Avertissements :

- **Responsabilité** : un modèle abliterated ne filtre plus rien — l'usage reste soumis à la loi et à la charte de l'entreprise ; ne jamais le servir à des tiers ni l'utiliser en production.
- **Qualité** : comparer avec le modèle de base sur vos tâches réelles ; certains dépôts abliterated dégradent le raisonnement.
- **Licence** : celle du modèle de base (Gemma = conditions Gemma, Qwen = Apache 2.0...).

## Télécharger un modèle sans passer par Hugging Face

Contexte : le pare-feu d'entreprise bloque `huggingface.co` et `cdn-lfs.huggingface.co`, mais pas les outils génériques (`curl`, `wget`, `aria2c`, `rclone`, navigateur vers Drive...). Plusieurs voies :

### Option 1 : le catalogue LM Studio (ne touche pas à HF)

Les modèles du catalogue LM Studio (`lmstudio-community`, modèles "staff picks") sont servis par le CDN de LM Studio (`files.lmstudio.ai`), pas par Hugging Face. Dans l'app, chercher le modèle et télécharger ; en CLI :

```bash
lms get lmstudio-community/gemma-4-12B-it-QAT-GGUF
```

`lms get` sait aussi résoudre des URL d'artefacts directes (pas seulement HF) : si un hébergeur de fichiers est autorisé par votre pare-feu, pointer dessus directement.

### Option 2 : miroir de Hugging Face

```bash
export HF_ENDPOINT=https://hf-mirror.com
lms get unsloth/Qwen3-8B-GGUF
# ou avec la CLI HF :
hf download unsloth/Qwen3-8B-GGUF --include "*Q4_K_M*"
```

Dans l'app : Paramètres → Téléchargements → URL de miroir. Autres miroirs/registres souvent non bloqués : **ModelScope** (`modelscope.cn`, `pip install modelscope && modelscope download --model <repo>`), **Kaggle** ([kaggle.com/models](https://www.kaggle.com/models), ex. `google/gemma-4`), la bibliothèque **Ollama** si l'outil est autorisé.

### Option 3 : transit par le drive (workflow hors ligne)

Pour un poste où seuls les outils standards passent :

1. **Sur une machine avec accès** (perso, 4G...) : télécharger le GGUF avec `aria2c -x 16 -s 16 <url-resolve>` ou `hf download <repo> --include "*Q4_K_M*"`, ou directement dans le navigateur.
2. **Uploader sur Drive** : interface web, ou `rclone` (une seule config initiale : `rclone config` → remote `gdrive`) :

   ```bash
   rclone copy ~/Téléchargements/Qwen3-8B-Q4_K_M.gguf gdrive:models/
   ```

3. **Sur le poste pro** : télécharger depuis Drive (`rclone copy gdrive:models/Qwen3-8B-Q4_K_M.gguf ~/Téléchargements/` ou l'interface web), puis vérifier l'intégrité (taille, et `sha256sum` si le dépôt publie un checksum) — un transfert multi-Go peut corrompre un fichier.
4. **Importer dans LM Studio** au choix :

   ```bash
   # CLI (expérimental)
   lms import ~/Téléchargements/Qwen3-8B-Q4_K_M.gguf

   # ou déposer le fichier dans l'arborescence attendue (scannée au lancement)
   mkdir -p ~/.cache/lm-studio/models/unsloth/Qwen3-8B-GGUF
   cp ~/Téléchargements/Qwen3-8B-Q4_K_M.gguf ~/.cache/lm-studio/models/unsloth/Qwen3-8B-GGUF/

   # ou, dans l'app : Paramètres → Modèles → ajouter un dossier de modèles personnalisé
   lms ls   # vérifier que le modèle est listé
   ```

Limites : le Drive gratuit fait 15 Go — prévoir **un modèle à la fois** (un Q4 de 5-9 Go passe, un 17-19 Go non) et nettoyer derrière soi.

Règle de sécurité : ne télécharger que des dépôts connus (éditeur officiel, `unsloth`, `bartowski`, `lmstudio-community`, `huihui-ai`...) et vérifier le checksum quand il est publié. Un miroir tiers, c'est un hébergeur auquel vous faites confiance avec des poids exécutés sur votre machine.

## Serveur API local

Pour exposer le modèle à d'autres applications :

1. Onglet **Developer** → activer **Start Server**. L'endpoint est `http://localhost:1234/v1`, protocole OpenAI. Par défaut seul le localhost y accède ; activer « Serve on Local Network » pour le réseau local.
2. Définir un jeton API dans les paramètres du serveur si autre chose que le localhost doit y accéder.

Test :

```bash
curl http://localhost:1234/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-8b",
    "messages": [{"role": "user", "content": "Bonjour !"}]
  }'
```

Lister ce qui est servi (seul le modèle **chargé** répond) :

```bash
curl -s http://localhost:1234/v1/models | jq -r '.data[].id'
```

N'importe quel SDK OpenAI (Python, LangChain, Continue.dev...) fonctionne en pointant `base_url` vers `http://localhost:1234/v1`.

## Usage agentique avec OpenCode

OpenCode est un agent de codage en terminal qui parle à n'importe quelle API compatible OpenAI — donc au serveur LM Studio. L'usage agentique exige un modèle avec **tool calling**.

### Modèles adaptés à l'agentique sur cette machine (32 Go, CPU)

| Modèle | Taille Q4 | Pourquoi |
|--------|-----------|----------|
| Qwen3 8B | ~5 Go | Tool calling natif, contexte 32K natif, rapide sur CPU — le choix par défaut pour un agent local |
| Gemma 4 12B | ~7 Go | Tool use + vision natifs (l'agent peut lire des captures), contexte natif 256K (à limiter en pratique) |
| Qwen3-Coder 30B-A3B | ~19 Go | Spécialisé code, MoE (~3B params actifs), contexte 256K — voir l'article [ollama](../ollama/README.md) pour le détail ; utilisable mais lent sur CPU |

À éviter pour l'agentique : les modèles sans tool calling (l'agent boucle sans appeler d'outils) et les denses > 20 Go (3-6 tok/s sur CPU, agent inutilisable).

### Connexion

1. Dans LM Studio : charger le modèle, onglet **Developer** → **Start Server** (port 1234).
2. Noter l'identifiant servi : `curl -s http://localhost:1234/v1/models` (ex. `qwen/qwen3-8b`). On peut le raccourcir au chargement : `lms load qwen/qwen3-8b --identifier qwen3-8b`.
3. Configurer OpenCode dans `~/.config/opencode/opencode.json` (global) ou `opencode.json` à la racine du projet :

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "lmstudio": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "LM Studio (local)",
      "options": {
        "baseURL": "http://127.0.0.1:1234/v1",
        "apiKey": "lmstudio"
      },
      "models": {
        "qwen/qwen3-8b": {
          "name": "Qwen3 8B (LM Studio)",
          "limit": { "context": 32768, "output": 8192 }
        },
        "google/gemma-4-12b-qat": {
          "name": "Gemma 4 12B (LM Studio)",
          "limit": { "context": 32768, "output": 8192 }
        }
      }
    }
  }
}
```

- `apiKey` : valeur bidon, le serveur local n'exige aucune clé (sauf si vous en avez défini une).
- `limit.context` est **à déclarer impérativement** (valeur = le contexte réellement servi par LM Studio) : sinon OpenCode ne sait pas quand compacter la conversation.
- Lancer : `opencode` puis `/models`, ou directement `opencode run --model lmstudio/qwen/qwen3-8b "ta tâche"`.

### Réglages pour le CPU 32 Go

- Dans LM Studio, régler la **longueur de contexte** du modèle chargé à 8192-32768 (chaque token de cache KV occupe de la RAM ; 256K est impossible ici).
- Sur Qwen3, désactiver le mode thinking pour les tâches simples (plus rapide) : dans le chat LM Studio ou via le chat template (`enable_thinking: false`).
- Un seul modèle chargé à la fois ; surveiller `free -h` pendant une session agentique.

## Déploiement Docker

Pour intégrer LM Studio à un parc de conteneurs, deux voies :

### Option A : `lmstudio/llmster-preview` (headless, CPU-only) — recommandée pour l'API

Image officielle du daemon headless, adaptée aux machines sans GPU dédié (statut *Technical Preview*).

```yaml
# docker-compose.yml
services:
  lmstudio:
    image: lmstudio/llmster-preview:latest
    ports:
      - "1234:1234"
    volumes:
      - ./lmstudio-models:/root/.lmstudio/models
    restart: unless-stopped
```

```bash
docker compose up -d
curl http://localhost:1234/v1/models
```

Les modèles peuvent être déposés directement dans le volume (`./lmstudio-models`) au format GGUF (utile avec le workflow drive de la section précédente), ou gérés via la CLI `lms`.

### Option B : `lscr.io/linuxserver/lm-studio` (GUI complète dans le navigateur)

Image communautaire (LinuxServer.io) qui streame l'interface graphique dans le navigateur.

```yaml
services:
  lm-studio:
    image: lscr.io/linuxserver/lm-studio:latest
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
    volumes:
      - ./config:/config
    ports:
      - "3000:3000"
      - "3001:3001"
    shm_size: "1gb"
    restart: unless-stopped
```

Interface accessible sur `https://localhost:3000` (certificat auto-signé).

### Quel choix pour cette machine

- **Option A** pour servir l'API à vos autres conteneurs : légère, headless, réutilisable dans le réseau Docker des projets.
- **Option B** si vous voulez garder la GUI complète sans installation native.
- Hybride pragmatique : AppImage native pour la découverte et le test de modèles en GUI + Option A pour le service API permanent. Les deux peuvent partager le même dossier de modèles monté en volume.

Limites à connaître :

- L'image `llmster-preview` est en preview : épingler une version précise plutôt que `latest` en usage durable.
- Pas d'accélération GPU en conteneur sur cette machine (pas de GPU dédié) : les perfs conteneur ≈ perfs natives.
- Attention à la RAM : le conteneur + l'OS + vos apps partagent les 32 Go.

## Pièges connus

- **AppImage ne lance pas** : `libfuse2` manquant (voir installation).
- **Téléchargement bloqué par le pare-feu d'entreprise** (huggingface.co / cdn-lfs) : passer par le catalogue LM Studio (`lms get`), un miroir (`HF_ENDPOINT=https://hf-mirror.com`), ou le workflow drive + `lms import` — voir la section dédiée.
- **GGUF transféré par le drive corrompu** : comparer la taille, vérifier le `sha256sum` quand le dépôt publie un checksum, réimporter.
- **Drive gratuit = 15 Go** : un seul gros modèle à la fois ; nettoyer après usage.
- **Génération très lente** : modèle trop gros → swap. Vérifier `free -h` pendant l'inférence, descendre de taille.
- **Réponses incohérentes** : quantification trop agressive (Q2/Q3). Remonter à Q4_K_M.
- **Contexte par défaut court** : LM Studio charge souvent avec 4096 tokens par défaut ; ajuster selon le besoin et la RAM.
- **Deux modèles en même temps** : le second évince le premier — planifier le chargement dans les scripts ; le serveur ne répond que pour le modèle chargé.
- **Modèle abliterated qui déraille** : comparer avec le modèle de base, choisir un autre dépôt (huihui-ai vs Heretic vs IlyaGusev) ; ne jamais l'exposer à des tiers.
- **OpenCode sans tool calling** : l'agent ne peut pas appeler d'outils et boucle ou échoue — choisir un modèle de la table agentique.
- **`limit.context` absent dans `opencode.json`** : compaction de conversation déclenchée au mauvais moment — déclarer le contexte réellement servi.
- **Contexte énorme demandé à OpenCode sur CPU** : le cache KV fait gonfler la RAM jusqu'au swap — rester à 8-32K sur cette machine.
- **Conteneur sans limite mémoire** (`mem_limit`) : peut mettre la machine à genoux ; fixer des limites dans le compose.

## Références

- [Site officiel LM Studio](https://lmstudio.ai)
- [LM Studio — importer un modèle téléchargé hors de l'app](https://lmstudio.ai/docs/app/advanced/import-model)
- [LM Studio — `lms get` (catalogue LM Studio, dépôts HF, URL directes)](https://lmstudio.ai/docs/cli/local-models/get)
- [LM Studio headless (llmster) — documentation officielle](https://lmstudio.ai/docs/developer/core/headless)
- [Image Docker officielle llmster-preview](https://hub.docker.com/r/lmstudio/llmster-preview)
- [Image LinuxServer.io lm-studio](https://docs.linuxserver.io/images/docker-lm-studio/)
- [Catalogue de modèles LM Studio](https://lmstudio.ai/models) (servi par `files.lmstudio.ai`, sans passer par Hugging Face)
- [Miroir Hugging Face hf-mirror.com](https://hf-mirror.com) (`HF_ENDPOINT=https://hf-mirror.com`)
- [ModelScope](https://modelscope.cn) et [Kaggle Models](https://www.kaggle.com/models) — registres de poids alternatifs
- [rclone — configurer un remote Google Drive](https://rclone.org/drive/)
- [Qwen3-8B GGUF (unsloth)](https://huggingface.co/unsloth/Qwen3-8B-GGUF)
- [Gemma 4 — documentation Google](https://ai.google.dev/gemma/docs/core) et [Gemma 4 dans LM Studio](https://lmstudio.ai/models/gemma-4)
- [Gemma 4 12B GGUF (unsloth)](https://huggingface.co/unsloth/gemma-4-12b-it-GGUF)
- [Modèles abliterated sur Hugging Face (filtre `abliterated`)](https://huggingface.co/models?other=abliterated)
- [huihui-ai — organisation de référence pour les modèles abliterated](https://huggingface.co/huihui-ai)
- [IlyaGusev/gemma-2-9b-it-abliterated](https://huggingface.co/IlyaGusev/gemma-2-9b-it-abliterated) (méthode originale)
- [bartowski/gemma-2-9b-it-abliterated-GGUF](https://huggingface.co/bartowski/gemma-2-9b-it-abliterated-GGUF)
- [Guide des modèles abliterated 2026 (locallyuncensored.com)](https://locallyuncensored.com/blog/abliterated-models-guide.html)
- [OpenCode — providers (dont LM Studio)](https://opencode.ai/docs/providers/)
- [Modèles Qwen GGUF sur Hugging Face](https://huggingface.co/Qwen)
