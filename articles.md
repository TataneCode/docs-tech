# Docs tech — tous les articles

> Fichier généré le 2026-11-26 à partir des 20 fichiers Markdown du répertoire `articles/` (9 sujets), dans l'ordre du sommaire du `README.md` racine. Sujets et chapitres sont séparés par `---`. Les sources à jour restent les fichiers d'origine dans `articles/<sujet>/` ; les liens internes y pointent désormais directement.

## Sommaire

- [LM Studio — héberger des LLM en local sur Ubuntu (ThinkPad T14, 32 Go en CPU)](#lm-studio--héberger-des-llm-en-local-sur-ubuntu-thinkpad-t14-32-go-en-cpu)
- [Shell Linux — cheat sheet développeur](#shell-linux--cheat-sheet-développeur)
- [Vim — cheat sheet développeur](#vim--cheat-sheet-développeur)
- [colibri — synthèse et mise en place d'un modèle de développement (Qwen3-Coder-30B-A3B)](#colibri--synthèse-et-mise-en-place-dun-modèle-de-développement-qwen3-coder-30b-a3b)
  - [colibri en C — intérêt d'un fork en Rust](#colibri-en-c--intérêt-dun-fork-en-rust)
- [ollama — modèle de codage local sur CPU (32 Go) avec OpenCode](#ollama--modèle-de-codage-local-sur-cpu-32-go-avec-opencode)
- [FreeLLMAPI — agréger les free tiers LLM et connecter OpenCode](#freellmapi--agréger-les-free-tiers-llm-et-connecter-opencode)
- [LLM — fondamentaux et panorama](#llm--fondamentaux-et-panorama)
  - [Chapitre 1 — Fonctionnement d'un LLM](#chapitre-1--fonctionnement-dun-llm)
  - [Chapitre 2 — L'entraînement d'un LLM](#chapitre-2--lentraînement-dun-llm)
  - [Chapitre 3 — Les hallucinations](#chapitre-3--les-hallucinations)
  - [Chapitre 4 — Température et paramètres de génération](#chapitre-4--température-et-paramètres-de-génération)
  - [Chapitre 5 — Le prompt engineering](#chapitre-5--le-prompt-engineering)
  - [Chapitre 6 — Fine-tuning et RAG](#chapitre-6--fine-tuning-et-rag)
  - [Chapitre 7 — Créer un agent IA](#chapitre-7--créer-un-agent-ia)
  - [Chapitre 8 — Bilan des modèles et de leur utilisation](#chapitre-8--bilan-des-modèles-et-de-leur-utilisation)
  - [Chapitre 9 — Comment fonctionne un réseau de neurones](#chapitre-9--comment-fonctionne-un-réseau-de-neurones)
  - [Chapitre 10 — L'inférence : faire tourner un LLM](#chapitre-10--linférence--faire-tourner-un-llm)
- [Harness engineering — concevoir l'environnement d'un agent IA](#harness-engineering--concevoir-lenvironnement-dun-agent-ia)
- [MCP — Model Context Protocol : fonctionnement du protocole et clients CLI/TUI (Inspector, mcp-tui)](#mcp--model-context-protocol--fonctionnement-du-protocole-et-clients-clitui-inspector-mcp-tui)

---

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
| Qwen3-Coder 30B-A3B | ~19 Go | Spécialisé code, MoE (~3B params actifs), contexte 256K — voir l'article [ollama](articles/ollama/README.md) pour le détail ; utilisable mais lent sur CPU |

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

---

# Shell Linux — cheat sheet développeur

> Catégorie : Cheat sheet

**Date :** 2026-10-18
**Statut :** Brouillon

## Objectif

Retrouver en quelques secondes les commandes shell du quotidien de développeur : permissions et exécution, recherche dans les logs, redirections de sortie — orienté recherche rapide, pas tutoriel.

## Prérequis

- Un terminal Linux (base Debian/Ubuntu testée)
- Aucun prérequis de niveau

## Navigation et fichiers

| Commande | Rôle | Exemple |
|----------|------|---------|
| `ls -lah` | Lister avec fichiers cachés, tailles lisibles | `ls -lah /var/log` |
| `cd -` | Retourner au dossier précédent | `cd -` |
| `pwd` | Afficher le dossier courant | `pwd` |
| `find` | Chercher des fichiers | `find . -name "*.log"` |
| `find` (taille) | Fichiers > 100 Mo | `find . -size +100M` |
| `tree -L 2` | Arborescence sur 2 niveaux | `tree -L 2` |
| `mkdir -p` | Créer une arborescence complète | `mkdir -p a/b/c` |

## Permissions et exécution

| Commande | Rôle | Exemple |
|----------|------|---------|
| `chmod +x` | Rendre un fichier exécutable | `chmod +x script.sh` |
| `chmod` (octal) | Fixer des droits précis | `chmod 755 script.sh` |
| `chmod -R` | Récursif sur un dossier | `chmod -R 755 bin/` |
| `./script.sh` | Exécuter le script du dossier courant | `./script.sh` |
| `chown` | Changer le propriétaire | `sudo chown user:groupe file` |
| `ls -l` | Voir les droits | `ls -l script.sh` |

Notation octale : lecture=4, écriture=2, exécution=1. `755` = propriétaire tout, groupe et autres lecture+exécution. `644` = standard pour un fichier non exécutable.

## Recherche dans les fichiers (grep)

| Commande | Rôle | Exemple |
|----------|------|---------|
| `grep motif` | Chercher dans un fichier | `grep ERROR app.log` |
| `grep -n` | Avec numéros de ligne | `grep -n ERROR app.log` |
| `grep -i` | Ignorer la casse | `grep -i error app.log` |
| `grep -r` | Récursif dans un dossier | `grep -rn "TODO" src/` |
| `grep -v` | Inverser (exclure le motif) | `grep -v DEBUG app.log` |
| `grep -E` | Regex étendue | `grep -E "ERROR|WARN" app.log` |
| `grep -c` | Compter les occurrences | `grep -c ERROR app.log` |
| `tail -f` | Suivre un log en direct | `tail -f app.log` |
| `tail -n 100` | 100 dernières lignes | `tail -n 100 app.log` |
| `head -n 20` | 20 premières lignes | `head -n 20 app.log` |
| `wc -l` | Compter les lignes | `grep ERROR app.log \| wc -l` |
| `find + grep` | Grep sur des fichiers trouvés | `find . -name "*.log" -exec grep -l ERROR {} +` |

## Redirections et pipes

| Syntaxe | Rôle | Exemple |
|---------|------|---------|
| `>` | Écraser la sortie dans un fichier | `./app > out.txt` |
| `>>` | Ajouter à la fin d'un fichier | `./app >> out.txt` |
| `2>` | Rediriger les erreurs | `./app 2> err.txt` |
| `2>&1` | Erreurs vers la même sortie | `./app > out.txt 2>&1` |
| `2>/dev/null` | Ignorer les erreurs | `./app 2>/dev/null` |
| `tee` | Afficher ET écrire dans un fichier | `./app \| tee out.txt` |
| `tee -a` | Idem, en ajout | `./app \| tee -a out.txt` |
| `<` | Lire l'entrée depuis un fichier | `./app < input.txt` |
| `\|` | Chaîner les commandes | `ps aux \| grep nginx` |

## Processus et disque

| Commande | Rôle | Exemple |
|----------|------|---------|
| `ps aux` | Lister les processus | `ps aux \| grep python` |
| `pgrep` | PID d'un processus par nom | `pgrep -f "python app.py"` |
| `kill` | Terminer un processus | `kill 1234` |
| `kill -9` | Forcer (dernier recours) | `kill -9 1234` |
| `pkill` | Tuer par nom | `pkill -f "python app.py"` |
| `df -h` | Espace disque par partition | `df -h` |
| `du -sh` | Taille d'un dossier | `du -sh ./logs` |
| `htop` | Moniteur interactif | `htop` |

## One-liners utiles

```bash
# Nombre d'erreurs dans un log
grep -c ERROR app.log

# Erreurs avec contexte : 2 lignes avant/après
grep -B2 -A2 ERROR app.log

# Trier les erreurs par fréquence (top motifs)
grep ERROR app.log | sort | uniq -c | sort -rn | head

# PID d'un port occupé
lsof -i :8080

# Vider un fichier sans le supprimer (utile pour un log ouvert)
> app.log

# Exécuter et journaliser tout (sortie + erreurs) horodaté
./app 2>&1 | tee "$(date +%F-%T).log"
```

## Pièges connus

- `>` écrase sans confirmation — préférer `>>` en cas de doute.
- `./app 2>&1 > out.txt` ne fait pas ce qu'on croit : l'ordre compte, écrire `./app > out.txt 2>&1`.
- `grep motif dossier/` sans `-r` cherche les fichiers du dossier en texte, pas à l'intérieur ; utiliser `grep -r`.
- `kill -9` en premier réflexe empêche le process de nettoyer derrière lui ; tenter `kill` simple d'abord.
- Un script `.sh` non exécutable : `bash script.sh` marche, mais vérifier le `chmod +x` et le shebang (`#!/bin/bash`) pour l'exécution directe.

## Références

- [man bash](https://www.gnu.org/software/bash/manual/bash.html)
- [GNU grep — documentation](https://www.gnu.org/software/grep/manual/grep.html)
- [Coreutils (ls, cp, tee, sort, wc...)](https://www.gnu.org/software/coreutils/manual/)

### Apprendre et pratiquer le shell (100 % gratuit, à suivre dans cet ordre)

- [Linux Journey](https://linuxjourney.com/) — cours gratuits et progressifs pour apprendre Linux pas à pas : choix de la distribution, ligne de commande, navigation dans les fichiers, réseau, administration système.
- [Terminus (jeu du MIT)](https://web.mit.edu/mprat/Public/web/Terminus/Web/main.html) — jeu d'aventure textuel jouable dans le navigateur où chaque action est une vraie commande shell (`ls`, `cd`, `cat`, `man`...) : les lieux du jeu sont des répertoires.
- [Linux Survival](https://linuxsurvival.com/) — tutoriel interactif gratuit avec un terminal Linux simulé dans le navigateur (prompt `zoo>`) pour pratiquer la douzaine de commandes essentielles sans installer Linux.
- [OverTheWire — Bandit](https://overthewire.org/wargames/bandit/) — wargame par niveaux (0 à 34) à jouer en SSH sur `bandit.labs.overthewire.org:2220`, conçu pour les grands débutants : chaque niveau fait découvrir des commandes et des fichiers Unix réels (mots de passe cachés, permissions, `grep`, `find`...).

---

# Vim — cheat sheet développeur

> Catégorie : Cheat sheet

**Date :** 2026-10-18
**Statut :** Brouillon

## Objectif

Consulter et éditer rapidement des fichiers et du code avec Vim : ouvrir les fichiers créés par redirections shell, chercher une erreur dans un log, éditer du code — sans lire un tutoriel complet.

## Prérequis

- Vim installé : `sudo apt install vim`
- Connaître les trois modes (ci-dessous) suffit pour toute la feuille

## Les modes : la clé de Vim

| Mode | Rôle | Comment y entrer | Comment en sortir |
|------|------|------------------|-------------------|
| Normal | Navigation et commandes | `Échap` (touche refuge) | — |
| Insertion | Taper du texte | `i`, `a`, `o` | `Échap` |
| Commande | Sauvegarder, quitter, chercher | `:` depuis Normal | `Échap` |

Réflexe d'or : **en cas de doute, taper `Échap`** pour revenir en mode Normal.

## Survie absolue

| Touche | Action |
|--------|--------|
| `:w` | Sauvegarder |
| `:q` | Quitter |
| `:wq` | Sauvegarder et quitter |
| `:q!` | Quitter sans sauvegarder (abandonner les modifs) |
| `:w nom.txt` | Sauvegarder sous un autre nom |
| `Échap` | Retour au mode Normal en toute circonstance |

## Ouvrir des fichiers

| Commande | Rôle |
|----------|------|
| `vim app.log` | Ouvrir (ou créer) un fichier |
| `vim -R app.log` | Ouvrir en lecture seule |
| `vim -d a.txt b.txt` | Comparer deux fichiers (diff) |
| `vim src/ script.sh` | Ouvrir plusieurs fichiers (`:n` suivant, `:N` précédent) |
| `view app.log` | Alias lecture seule |

## Navigation

| Touche | Action |
|--------|--------|
| `h j k l` | Gauche, bas, haut, droite |
| `w` / `b` | Mot suivant / précédent |
| `0` / `$` | Début / fin de ligne |
| `gg` / `G` | Début / fin du fichier |
| `42G` ou `:42` | Aller à la ligne 42 |
| `{` / `}` | Paragraphe précédent / suivant |
| `Ctrl-d` / `Ctrl-u` | Descendre / montrer d'un demi-écran |
| `Ctrl-e` / `Ctrl-y` | Scroller d'une ligne |
| `zz` | Centrer l'écran sur le curseur |

## Édition

| Touche | Action |
|--------|--------|
| `i` / `a` | Insérer avant/après le curseur |
| `o` / `O` | Nouvelle ligne en dessous/dessus |
| `x` | Supprimer un caractère |
| `dd` | Couper la ligne |
| `yy` | Copier la ligne |
| `p` / `P` | Coller après/avant |
| `u` | Annuler |
| `Ctrl-r` | Rétablir |
| `r` | Remplacer un caractère |
| `cw` | Changer le mot sous le curseur |
| `d$` | Couper jusqu'à la fin de ligne |
| `.` | Répéter la dernière action |
| `3dd` | Couper 3 lignes (compteur applicable partout) |

## Recherche et substitution

| Commande | Rôle |
|----------|------|
| `/motif` | Chercher en avant |
| `?motif` | Chercher en arrière |
| `n` / `N` | Occurrence suivante / précédente |
| `*` | Chercher le mot sous le curseur |
| `:%s/ancien/nouveau/g` | Remplacer dans tout le fichier |
| `:%s/ancien/nouveau/gc` | Idem avec confirmation à chaque fois |
| `:s/ancien/nouveau/` | Remplacer sur la ligne courante |

## Pour le développeur

| Commande | Rôle |
|----------|------|
| `:set number` / `:set nu` | Numéros de ligne |
| `:set relativenumber` | Numérotation relative (pratique pour `5j`, `3dd`) |
| `>>` / `<<` | Indenter / désindenter la ligne |
| `=%` | Réindenter le bloc sous le curseur |
| `:sp` / `:vsp` | Split horizontal / vertical |
| `Ctrl-w w` | Basculer entre splits |
| `:e fichier` | Ouvrir un autre fichier |
| `Ctrl-o` / `Ctrl-i` | Position précédente / suivante dans l'historique |
| `:w !sudo tee %` | Sauvegarder un fichier ouvert sans sudo (le piège classique) |

## Config minimale — `~/.vimrc`

```vim
set number
set relativenumber
syntax on
set tabstop=4
set shiftwidth=4
set expandtab
set incsearch
set hlsearch
```

À copier tel quel pour une expérience correcte en édition de code.

## Séquences types

```text
Ouvrir un log généré par redirection et chercher une erreur :
  vim app.log  →  /ERROR  →  n (occurrences suivantes)  →  :q

Éditer un script :
  vim script.sh  →  /motif  →  cw + nouveau texte  →  Échap  →  :wq

Reformater tous les "foo" en "bar" dans un fichier :
  vim file.txt  →  :%s/foo/bar/g  →  :wq
```

## Pièges connus

- **Impossible de quitter** : `Échap` puis `:q!` (abandonner) ou `:wq` (sauvegarder). C'est le piège n°1.
- **Taper du texte en mode Normal** produit des comportements étranges (suppressions, sauts) : taper `Échap` puis `u` pour annuler, puis `i` pour insérer proprement.
- `:q` refuse de quitter si des modifications non sauvegardées existent : c'est voulu, utiliser `:wq` ou `:q!`.
- Coller du texte avec `Ctrl-V` ne fonctionne pas dans Vim : en mode Insertion, le collage terminal (`Ctrl-Shift-V`) insère parfois avec indentation cassée — activer `:set paste` avant de coller, et `:set nopaste` après.
- Sur une machine minimale (conteneur), `vim` peut être absent : `vi` y est souvent en version minimale ; les commandes de base ci-dessus fonctionnent pareil.

## Références

- [Vim — documentation officielle](https://www.vim.org/docs.php)
- `vimtutor` — tutoriel interactif de 30 min, installé avec Vim
- [Learn Vimscript the Hard Way](https://learnvimscriptthehardway.stevelosh.com/)

---

# colibri — synthèse et mise en place d'un modèle de développement (Qwen3-Coder-30B-A3B)

> Catégorie : Guide d'installation

**Date :** 2026-10-29 (affiné le 2026-10-31)
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
- `git`, **Python 3.10+** (obligatoire : le launcher `coli` et la passerelle API sont des scripts Python), un compilateur C (`gcc`/`clang` + OpenMP) ou le téléchargement du binaire précompilé
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

## Installer la CLI `coli` (non native)

`coli` n'est pas un binaire système livré par l'OS : c'est un **script Python** (`c/coli`, shebang `#!/usr/bin/env python3`) qui pilote les moteurs C compilés et la passerelle API. Sans Python 3.10+, rien ne démarre. Quatre façons de le rendre appelable depuis n'importe quel terminal, du plus simple au plus « propre » :

### 1. Depuis le dépôt cloné (ce que fait le setup one-step)

Après `git clone` + `./start-here.sh`, la CLI s'appelle **par son chemin** :

```bash
cd ~/colibri
c/coli status        # ou ./c/coli status
c/coli chat --model ~/colibri-models/<dossier>
c/coli serve --model ~/colibri-models/<dossier> --cap 128
```

`start-here.sh` fait exactement `python3 c/coli setup "$@"` en interne. C'est suffisant tant qu'on travaille dans le dossier du dépôt, mais `coli` n'est alors pas dans le `PATH`.

### 2. Lien symbolique dans `~/.local/bin` (recommandé, sans sudo)

Rendre `coli` appelable de partout **sans rien installer system-wide** :

```bash
mkdir -p ~/.local/bin
ln -s ~/colibri/c/coli ~/.local/bin/coli
coli --version       # vérifie : affiche « colibri v2.0.0 »
```

`~/.local/bin` est dans le `PATH` par défaut sur Ubuntu/WSL. Le launcher se localise lui-même via `realpath(__file__)` : même à travers le lien, il retrouve `c/` (moteurs, `tools/`, modules Python) à côté de lui. **Ne pas copier** le script ailleurs : copié hors de `c/`, il chercherait les moteurs dans `../libexec/colibri` et échouerait.

Sur **Windows natif**, l'équivalent est le lanceur `c\coli.cmd` (à appeler tel quel, ou à ajouter au `PATH`).

### 3. Installation « propre » via `make install` (préfixe `/usr/local`)

Après avoir compilé les moteurs (`make qwen36` dans `c/`, ou le setup one-step qui le fait) :

```bash
cd ~/colibri
sudo make install        # PREFIX=/usr/local par défaut
coli --version           # /usr/local/bin/coli est dans le PATH
```

Le Makefile (`c/Makefile`, `PREFIX ?= /usr/local`) installe alors :

- `$(PREFIX)/bin/coli` — le launcher, dans le `PATH` ;
- `$(PREFIX)/libexec/colibri/` — les 13 moteurs compilés (`colibri`, `qwen36`, `glm53`…), les modules Python (`openai_server.py`, `doctor.py`…) et `tools/`.

Le launcher bascule automatiquement sur ce layout installé (il détecte l'absence de binaire à côté de lui et cherche dans `../libexec/colibri`). Un préfixe utilisateur évite le `sudo` : `make install PREFIX=~/.local` (`~/.local/bin` est dans le `PATH` par défaut sur Ubuntu).

### 4. Via pip (`pip install -e .` depuis le dépôt)

Le dépôt embarque un `pyproject.toml` (paquet `colibri-engine`) qui déclare un point d'entrée console `coli = "colibri.cli:main"` : un mince wrapper Python qui délègue à `c/coli`. Depuis un clone :

```bash
cd ~/colibri
pip install -e .         # ou pipx install -e .
coli --version
```

Utile dans un venv dédié ; à préférer au lien symbolique si vous gérez déjà vos outils Python avec pip/pipx.

### Ordre conseillé avec le démarrage d'une instance Qwen3-Coder

Le setup one-step (`./start-here.sh --model qwen3-coder-30b`) **démarre déjà** `coli serve` en arrière-plan (port 8000) à la fin de l'installation. Pour piloter ensuite cette instance depuis un autre terminal, il faut que `coli` soit dans le `PATH` — d'où l'intérêt de faire l'étape 2 (lien symbolique) **avant ou juste après** le premier `./start-here.sh` :

```bash
# 1. Clone
git clone https://github.com/JustVugg/colibri
cd colibri

# 2. La CLI dans le PATH (une fois pour toutes)
mkdir -p ~/.local/bin && ln -s "$PWD/c/coli" ~/.local/bin/coli
coli --version

# 3. Setup + démarrage de l'instance (Qwen3-Coder, 19,4 Go téléchargés)
./start-here.sh --model qwen3-coder-30b

# 4. Depuis un autre terminal : piloter l'instance
coli status
coli stop
coli serve --model ~/colibri-models/<dossier> --cap 128 --port 8000   # relance manuelle
```

Récapitulatif : `./start-here.sh` fait le **premier** démarrage ; la CLI `coli` dans le `PATH` sert à **piloter** l'instance ensuite (`status`, `stop`, `logs`, relance manuelle de `serve`/`chat` avec d'autres options comme `--cap 128`).

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

Relancer ensuite : `./start-here.sh` démarre directement. `./start-here.sh --list` montre tous les modèles compatibles avec la machine. `coli status` / `coli stop` pour voir/arrêter (avec `coli` rendu appelable partout, voir [Installer la CLI `coli`](#installer-la-cli-coli-non-native) ci-dessus).

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
- **CLI `coli` « commande introuvable »** : `coli` n'est livré par aucun paquet système. Depuis un clone, appelez `c/coli ...` ou posez le lien `~/.local/bin/coli -> ~/colibri/c/coli` (ne copiez pas le script hors de `c/`, il ne retrouverait pas ses moteurs). Le setup one-step (`start-here.sh`) démarre le serveur tout seul ; la CLI dans le `PATH` sert surtout à piloter l'instance (`status`, `stop`, `logs`, relance manuelle)
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
- [Makefile upstream](https://github.com/JustVugg/colibri/blob/main/c/Makefile) — cible `install` (`PREFIX`, `BINDIR`, `LIBEXECDIR`) qui pose `coli` dans le `PATH`
- [Launcher `c/coli`](https://github.com/JustVugg/colibri/blob/main/c/coli) — script Python pilote (layouts « dépôt cloné » et « installé » via `libexec`)
- [Serveur MCP](https://github.com/JustVugg/colibri/blob/main/docs/MCP_SERVER.md) — « après `make install`, le launcher est sur le `PATH` et `coli mcp` suffit »
- [Documentation opencode — providers](https://opencode.ai/docs/providers/) — provider personnalisé `@ai-sdk/openai-compatible`
- [Documentation opencode — agents](https://opencode.ai/docs/agents/) — section `agent` de `opencode.json`
- [Analyse : intérêt d'un fork en Rust](articles/colibri/fork-rust.md)

---

# colibri en C — intérêt d'un fork en Rust

> Catégorie : Notes & dépannage (réflexion d'architecture)

**Date :** 2026-10-28
**Statut :** Terminé

## Objectif

Évaluer si porter (forker) le moteur d'inférence C de [colibri](https://github.com/JustVugg/colibri) en Rust présente un intérêt réel, au-delà de l'attrait « langage moderne ».

## Contexte : ce qui fait la spécificité du code C

Avant de juger le portage, il faut regarder ce que le C fait réellement. Le code n'est pas du C générique : il est délibérément contraint pour des raisons de performance et de portabilité.

- **Zéro dépendance, C pur** : pas de BLAS, pas de framework, pas de runtime Python à l'exécution. Le Makefile compile chaque moteur avec un simple `gcc`/`clang` + OpenMP. C'est un choix de distribution (binaire statique, releases multi-plateformes, build sans outilchain particulier).
- **Un fichier `.c` par famille de modèles** (~29 fichiers `.c`, ~142 fichiers `.c`/`.h`/`.cu`/`.mm`, ~216 k lignes au total) au-dessus de **headers partagés** : `st.h` (safetensors), `quant.h` (décodeurs de conteneurs, noyaux entiers), `expert_ffn.h`/`expert_store.h` (noyau MoE et cache d'experts streamés), `tok.h`, `json.h`, `compat.h` (shims Windows/macOS).
- **Du code « unsafe by design » assumé** : buffers manuels, `pread` sur des offsets calculés, packing de nibbles int4, kernels SIMD écrits à la main, pools de threads maison pour le chargement des experts. La « politique de sécurité mémoire » du projet est la CI : chaque moteur est vérifié token par token contre l'implémentation de référence du modèle sur une fixture tiny (oracle teacher-forcing, `ORACLE_STRICT=1`).
- **Backends GPU optionnels et isolés** : CUDA (`backend_cuda.cu`, compilé par nvcc, avec un shim HIP via `backend_gpu_compat.h`), Metal (`backend_metal.mm`), Vulkan (`backend_vulkan.c` + `vk_tier.c`/`vk_chain.c`), XDNA. Le build par défaut reste CPU pur.
- **Politique stricte** (CONTRIBUTING.md) : `main` reste « known-good », PRs sur `dev`, build sans warning, oracle 30-32/32 en teacher-forcing + 20/20 greedy.

## Ce que Rust apporterait — les vrais gains

| Gain | Pourquoi c'est réel ici |
|---|---|
| **Sûreté mémoire par défaut** | Le domaine de colibri est exactement celui des bugs que Rust élimine : parsing de conteneurs binaires (safetensors, formats de quantification maison), indexation d'experts par offsets, buffers de prefetch concurrents. Un typo d'index ou un dépassement de buffer est le bug classique de ce genre de moteur — et coûte des heures de debug sur des modèles de 400 Go |
| **Concurrence sans data races** | Le moteur a déjà du parallélisme maison : pool de loaders d'experts, thread de prefetch routeur, batching continu sur 16 slots KV, overlap CPU/GPU. Tout ce code est du C avec des primitives manuelles (OpenMP, threads POSIX). Le modèle de possession/emprunt de Rust rend ce code **vérifiable** plutôt que « relu en PR » |
| **Itérateurs et zéro-cost abstractions** | Les boucles de décodage de tokens (route → union → place → overlap → learn) se prêtent bien à des itérateurs sûrs ; le compilateur génère le même code que le C bien écrit |
| **Écosystème** | `wgpu`/`vulkano` pour le backend Vulkan, `candle`-style kernels, `tiktoken`-rs, `safetensors` (crate officiel), `rayon` pour le parallélisme CPU, `memmap2` pour le streaming d'experts depuis le disque. Moins de code maison à maintenir |
| **Distribution** | Un binaire Rust statique reste un seul fichier, cross-compilable — le critère de distribution de colibri est préservé |
| **Testabilité** | Le projet a déjà une culture de tests (oracles, harnesses, fixtures tiny). Rust la renforce nativement (`cargo test`, fuzzing avec `cargo-fuzz`) |

## Ce que Rust coûterait — les vrais freins

| Frein | Détail |
|---|---|
| **Réécriture, pas traduction** | ~216 k lignes de C avec des noyaux SIMD/quantification finement ajustés. Un portage mécanique serait un désastre de performance ; une réécriture propre est un projet de **mois**, pas de semaines. Et la fidélité token-exacte (l'oracle) devrait être re-démontrée moteur par moteur |
| **Performance : parité difficile à prouver** | Le C de colibri est du C *optimisé à la main* (packing int4, noyaux entiers, prefetch logiciel). Rust peut atteindre la parité, mais chaque noyau critique demandera du `unsafe` + SIMD explicite (`std::simd`, `packed_simd` ou intrinsics) et des benchmarks systématiques. Le projet a une règle d'or : une optimisation ne vaut que par une mesure end-to-end reproductible |
| **Backends GPU** | CUDA en Rust existe (`rustacuda`, `cudarc`) mais est jeune ; Metal (`metal-rs`) et surtout le **HIP/ROCm via un shim unique** (le truc astucieux de `backend_gpu_compat.h`) seraient à reconstruire. Le backend Vulkan (`vulkano`/`wgpu`) est le plus accessible |
| **Windows/macOS** | Le C a des shims (`compat.h`) éprouvés ; en Rust il faudra réassembler la matrice de cibles (MSVC, MinGW, ARM64…) |
| **Communauté et rythme** | colibri est un projet actif (169 PRs sur la 2.0.0, 88 contributeurs externes). Un fork Rust diverge : il faut soit porter chaque amélioration upstream (coût permanent), soit upstreamiser (le projet C n'a pas demandé de Rust) |
| **Le « unsafe » ne disparaît pas** | Lecture de fichiers mappés, kernels SIMD, ABI avec les backends GPU : une grande partie du code restera `unsafe`. Rust réduit la surface, ne l'annule pas — le gain est réel mais pas magique |

## Verdict

**Oui, un fork Rust a un intérêt, mais ce n'est pas une évidence.**

L'intérêt est **réel et ciblé** :

1. **Sûreté** sur le parsing de conteneurs et l'indexation mémoire — le cœur du risque de ce type de moteur.
2. **Concurrence vérifiable** pour le streaming d'experts, le prefetch, le batching multi-slots — là où le C maison est le plus fragile.
3. **Maintenabilité** à long terme et onboarding de contributeurs (Rust est plus lisible pour du code unsafe maîtrisé que du C pointer-arithmetic).

Mais le coût est **élevé et structurel** :

- C'est une **réécriture de ~216 k lignes** avec re-validation token-exacte de 13 moteurs, pas un portage incrémental réaliste.
- La **performance brute** ne sera pas un argument de vente : le C actuel est déjà au niveau « noyaux manuels optimisés » ; Rust ne fera pas mieux sans le même travail d'optimisation, et devra le *prouver* par les mêmes benchmarks.
- La **divergence avec upstream** est le risque principal : colibri évolue vite (169 PRs sur une release).

### Quand le fork Rust se justifie

- Si l'objectif est **un moteur maintenable à long terme**, écrit pour durer et contribuer en amont (en proposant le portage au projet, pas en le forkant silencieusement).
- Si l'objectif est **la sûreté** (parsing de formats non fiables, exposition réseau via la passerelle) plutôt que la vitesse.
- Si l'on démarre **un moteur à partir de zéro** sur un sous-ensemble (ex. un seul moteur MoE + Vulkan via `vulkano`), en réutilisant les conteneurs et l'architecture — pas les 13 moteurs d'un coup.

### Quand il ne se justifie pas

- Si l'objectif est **de gagner en performance** : le C est déjà optimisé à la main ; le gain viendrait du travail d'optimisation, pas du langage.
- Si l'on veut **suivre upstream** : le coût de re-portage de chaque PR est prohibitif en l'état.
- Si l'on sous-estime l'effort : le vrai travail est la re-validation de la fidélité numérique, pas la traduction syntaxique.

### Piste pragmatique

Plutôt qu'un fork complet, une approche hybride est plus réaliste :

1. **Garder le C** pour les noyaux de calcul les plus chauds (ils sont déjà au plafond).
2. **Écrire en Rust** les couches périphériques à plus haut risque : parsing de conteneurs/quantification, passerelle API (aujourd'hui Python stdlib), launcher, prefetch/planification — là où la sûreté et la concurrence comptent le plus.
3. **Interfacer par FFI** (C ABI stable, `extern "C"`) : Rust appelle les noyaux C, ou l'inverse.
4. **Contribuer en amont** : le projet est ouvert (Apache 2.0, 88 contributeurs externes sur la dernière release) ; un composant Rust bien ciblé a plus de chances d'être accepté qu'un fork.

## Références

- [Dépôt GitHub colibri](https://github.com/JustVugg/colibri) — C pur, Apache 2.0, v2.0.0
- [CONTRIBUTING.md](https://github.com/JustVugg/colibri/blob/main/CONTRIBUTING.md) — politique de branches (`dev`), oracle CI, fidélité token-exacte
- [GPU_BACKENDS.md](https://github.com/JustVugg/colibri/blob/main/GPU_BACKENDS.md) — architecture des backends CUDA/HIP/Metal/Vulkan
- [Synthèse colibri et mise en place de GLM-5.3-Flash](articles/colibri/README.md)

---

# ollama — modèle de codage local sur CPU (32 Go) avec OpenCode

> Catégorie : Guide d'installation

**Date :** 2026-11-02
**Statut :** Terminé

## Objectif

Installer [Ollama](https://ollama.com) sur le **laptop Ubuntu 32 Go (Ryzen 7 PRO 4750U)** — machine sans GPU dédié, où l'inférence se fait sur **CPU + RAM** — et y faire tourner un **modèle de codage agentique** connecté à [OpenCode](https://opencode.ai) : choix du modèle (`qwen3-coder:30b` recommandé), réglage du contexte (`num_ctx`, point critique avec OpenCode), connexion d'OpenCode et surveillance de la RAM.

## Prérequis

- Laptop Ubuntu (24.04), **32 Go de RAM**, Ryzen 7 PRO 4750U (8 cœurs / 16 threads), SSD avec ~25 Go libres
- **Pas de GPU dédié** : Ollama tourne en CPU pur. L'iGPU (Arc sur les machines Core Ultra) et le **NPU ne sont pas exploités par Ollama** — `100% CPU` dans `ollama ps` est donc normal ici
- `curl` et les droits `sudo`
- Optionnel : OpenCode installé (voir [opencode.ai](https://opencode.ai))

## 1. Installer Ollama sur Ubuntu

Le script officiel installe le binaire, crée l'utilisateur système `ollama` et le service systemd :

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Vérifications :

```bash
ollama --version
sudo systemctl is-active ollama          # active (running)
curl -s http://localhost:11434/api/tags   # {"models":[]} au départ
```

- Le serveur écoute par défaut sur `127.0.0.1:11434` (localhost uniquement). Pour l'exposer au LAN : `OLLAMA_HOST=0.0.0.0:11434` — déconseillé sans pare-feu ni clé
- Mise à jour : le script ajoute le dépôt apt d'Ollama, donc `sudo apt update && sudo apt upgrade ollama` suffit

## 2. Choisir et tirer le modèle (32 Go de RAM, CPU uniquement)

| Rôle | Modèle | Taille disque | Pourquoi |
|---|---|---|---|
| **Recommandé** | `qwen3-coder:30b` | ~19 Go | MoE avec seulement **~3,3B paramètres actifs** par token, contexte natif 256K, tool calling natif. Sur CPU, c'est le meilleur rapport qualité/vitesse dans la tranche 24-32 Go : la faible part active le rend étonnamment fluide malgré sa taille. C'est le modèle le plus souvent recommandé pour le combo OpenCode + Ollama en 24-32 Go |
| Alternative | `gpt-oss:20b` | ~14 Go | MoE également, 128K de contexte, function calling natif, tourne dès 16 Go de RAM. Bon second choix si tu veux garder de la RAM libre |
| Filet de sécurité | `qwen2.5-coder:7b` | ~4,7 Go | Si le 30B ralentit la machine pendant une compilation ou Docker. Moins bon en agent, mais réactif |
| **À éviter** | denses 24-27B (Devstral Small 2, Qwen 27B dense…) | 14-17 Go | Ils tiennent en mémoire mais tombent à **3-6 tok/s sur CPU** → OpenCode inutilisable en mode agent |

```bash
ollama pull qwen3-coder:30b     # ~19 Go, le téléchargement est reprenable
ollama pull gpt-oss:20b         # optionnel
ollama pull qwen2.5-coder:7b    # optionnel
ollama list
ollama show qwen3-coder:30b --modelfile   # voir le num_ctx embarqué dans le modèle
```

Test rapide :

```bash
ollama run qwen3-coder:30b "Résume en une phrase ce que fait ce code : <colle du code>"
ollama ps                       # modèle chargé, taille en RAM, colonne PROCESSOR
```

## 3. Régler le contexte (`num_ctx`) — le point critique avec OpenCode

Par défaut, Ollama cale le contexte à **~4K tokens** (palier le plus bas quand aucune VRAM n'est détectée). Or OpenCode envoie **7k+ tokens dès le premier tour** (prompt système + outils) et recommande **64K+**. Sans réglage, le premier tour agent est tronqué ou échoue.

**Méthode A — pour tout le serveur** (variable `OLLAMA_CONTEXT_LENGTH`, lue au démarrage du serveur) :

```bash
sudo systemctl edit ollama
```

ajoute dans l'override :

```ini
[Service]
Environment="OLLAMA_CONTEXT_LENGTH=64000"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

**Méthode B — par modèle** (un Modelfile ; son `PARAMETER num_ctx` prend le dessus sur la variable d'environnement) :

```bash
cat > ~/Modelfile.qwen3-coder-64k <<'EOF'
FROM qwen3-coder:30b
PARAMETER num_ctx 65536
EOF
ollama create qwen3-coder-64k -f ~/Modelfile.qwen3-coder-64k
```

Variante interactive : `ollama run qwen3-coder:30b`, puis dans le chat `/set parameter num_ctx 65536` puis `/save qwen3-coder-64k`.

## 4. Connecter OpenCode

**Le plus simple** — Ollama génère la config OpenCode tout seul :

```bash
ollama launch opencode
```

**Config manuelle** si besoin, dans `~/.config/opencode/opencode.json` (global) ou `opencode.json` à la racine du projet — provider `@ai-sdk/openai-compatible` pointé sur `http://localhost:11434/v1` :

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": {
        "baseURL": "http://localhost:11434/v1",
        "apiKey": "local"
      },
      "models": {
        "qwen3-coder:30b": {
          "name": "Qwen3-Coder 30B (Ollama)",
          "limit": { "context": 65536, "output": 32768 }
        }
      }
    }
  }
}
```

- `apiKey: "local"` : valeur bidon, Ollama local n'exige aucune clé
- **`limit.context` est à déclarer impérativement** (valeur = le `num_ctx` réellement servi, ex. 65536) : sinon OpenCode ne sait pas quand compacter la conversation
- Vérification de l'endpoint : `curl -s http://localhost:11434/v1/models`
- Lancer : `opencode` puis `/models`, ou directement `opencode run --model ollama/qwen3-coder:30b "ta question"`

## 5. Surveiller la machine

```bash
ollama ps
free -h
```

- Colonne **PROCESSOR « 100% CPU » = normal** sur cette machine (pas de GPU dédié, iGPU/NPU non utilisés par Ollama). Le vrai signal d'alerte est la **RAM restante**
- Compter **poids du modèle + 20-30 % pour le contexte** : `qwen3-coder:30b` ≈ 19 Go + ~5 Go de contexte 64K ≈ **24 Go** utilisés
- Un seul gros modèle chargé à la fois ; pour décharger : `ollama stop qwen3-coder:30b`

## 6. Performances attendues

Attends-toi à du **correct sur des tâches petites et répétitives**, pas à un remplaçant de modèle cloud sur une migration multi-fichiers. Le MoE 30B reste fluide sur CPU grâce à sa faible part de paramètres actifs ; les denses 24-27B, eux, s'écroulent à 3-6 tok/s.

## Pièges connus

- **`num_ctx` par défaut ~4K** : premier tour OpenCode tronqué/échec → régler `OLLAMA_CONTEXT_LENGTH` (serveur) ou `num_ctx` (Modelfile), section 3
- `OLLAMA_CONTEXT_LENGTH` est lu **au démarrage du serveur** : après modification, `sudo systemctl restart ollama`
- Le `PARAMETER num_ctx` d'un Modelfile custom **écrase** la variable d'environnement serveur
- Changer `num_ctx` entre deux requêtes force un rechargement du modèle (latence ajoutée) : fixer une valeur et s'y tenir
- **NPU (Core Ultra) non exploité** par Ollama — ne pas compter dessus
- iGPU non utilisée → « 100% CPU » normal ; surveiller `free -h`, pas le CPU
- Ne pas charger plusieurs gros modèles en parallèle (`OLLAMA_MAX_LOADED_MODELS`) : la RAM part vite (modèle + 20-30 % de contexte chacun)

## Références

- [Ollama — installation Linux](https://ollama.com/download/linux) — script officiel `install.sh`
- [FAQ Ollama](https://docs.ollama.com/faq) — `OLLAMA_CONTEXT_LENGTH`, `num_ctx` par défaut ~4096, `OLLAMA_MAX_LOADED_MODELS`
- [Ollama × OpenCode](https://docs.ollama.com/integrations/opencode) — `ollama launch opencode`, config auto
- [Blog Ollama — `ollama launch`](https://ollama.com/blog/launch) — lance Claude Code / OpenCode / Codex sans fichier de config manuel
- [OpenCode + Ollama : setup et modèles](https://localaimaster.com/blog/opencode-ollama-setup) — `ollama launch opencode`, piège du contexte 4K, modèles 24-32 Go
- [Meilleurs LLM locaux pour le codage](https://modelfit.io/guides/best-local-llm-for-coding/) — qwen3-coder:30b recommandé en 24-32 Go
- [OpenCode avec modèles locaux — contexte 64K](https://jacar.es/en/opencode-with-local-models/)
- [Matrice agent-harness OpenCode](https://llmconfigurator.com/en/tools/agent-harness-matrix/opencode)
- [Playbook CLI coding agent OpenCode](https://build.nvidia.com/playbooks/cli-coding-agent/opencode)
- [Ollama + OpenCode — limites réelles](https://haimaker.ai/blog/ollama-opencode-setup/)
- [Modèles sur ollama.com](https://ollama.com/library) — `qwen3-coder`, `gpt-oss`, `qwen2.5-coder`
- [Documentation OpenCode — providers](https://opencode.ai/docs/providers/) — provider `@ai-sdk/openai-compatible`
- Article voisin : [colibri — Qwen3-Coder-30B-A3B en natif sur 32 Go](articles/colibri/README.md) (même modèle, moteur C pur sans Ollama)

---

# FreeLLMAPI — agréger les free tiers LLM et connecter OpenCode

> Catégorie : Guide d'installation

**Date :** 2026-10-28
**Statut :** Terminé

## Objectif

Installer [FreeLLMAPI](https://github.com/tashfeenahmed/freellmapi) en local (Docker), récupérer sa clé API unifiée, et connecter l'agent de code **OpenCode** à cette API via un provider custom OpenAI-compatible dans `opencode.json`.

## Prérequis

- Docker + Docker Compose (voie recommandée), ou Node.js >= 20.18 pour un lancement depuis les sources
- [OpenCode](https://opencode.ai) installé (`opencode --version`)
- Au moins une clé API gratuite chez un provider supporté (Google Gemini, Groq, Cerebras, Mistral, OpenRouter, Cloudflare, Cohere, Z.ai, NVIDIA, HuggingFace, ModelScope, etc. — 34 providers au total)

## Ce qu'est FreeLLMAPI

FreeLLMAPI est un routeur local-first, auto-hébergé et open source (MIT) qui agrège les free tiers d'une trentaine de fournisseurs de LLM derrière **un seul endpoint OpenAI-compatible** (`http://localhost:3001/v1`) :

- ~7,4 milliards de tokens/mois de capacité cumulée, 474 familles de modèles, 635 endpoints gratuits
- les clés fournisseurs sont chiffrées (AES-256-GCM) dans une base SQLite locale ; vos applications ne voient qu'une seule clé unifiée `freellmapi-…`
- routage intelligent (6 stratégies), bascule automatique sur le modèle suivant en cas de 429/5xx, suivi des quotas par clé (RPM/RPD/TPM/TPD)
- surfaces supportées : `/v1/chat/completions`, `/v1/responses`, `/v1/embeddings`, `/v1/images/generations`, `/v1/audio/speech`, `/v1/messages` (format Anthropic), `/v1beta` (format Gemini), émulation Ollama, serveur MCP
- catalogue de modèles auto-mis à jour depuis un flux signé (snapshot mensuel en gratuit, live en premium à $19/an)
- tableau de bord React à l'adresse `http://localhost:3001` (60 langues), application desktop (macOS/Windows), CLI `npx freellmapi`

Le routeur tourne en local : vos prompts vont de votre machine vers les fournisseurs que vous avez activés. Le serveur de catalogue ne voit ni prompts ni clés.

## Installation (Docker, one-liner)

```bash
curl -fsSL https://freellmapi.co/install.sh | bash
```

Le script crée `~/freellmapi`, génère une clé de chiffrement, tire l'image `ghcr.io/tashfeenahmed/freellmapi` et démarre le conteneur. Le relancer est sans danger (`.env` préservé, image mise à jour).

Alternative manuelle avec Docker Compose (depuis le dépôt cloné) :

```bash
git clone https://github.com/tashfeenahmed/freellmapi.git
cd freellmapi
docker compose up -d
```

## Mise en route

1. Ouvrir le tableau de bord : `http://localhost:3001` (compte email + mot de passe sur une install serveur).
2. Page **Keys** : ajouter au moins une clé API fournisseur (ex. une clé Google AI Studio gratuite).
3. Page **Models** : réordonner la chaîne de fallback (**Fallback Chain**) si besoin. Le modèle virtuel `auto` laisse le routeur choisir le meilleur modèle disponible.
4. Copier la **clé API unifiée** (`freellmapi-…`) dans l'en-tête de la page Keys — c'est le seul credential que vos clients utiliseront.

Test rapide :

```bash
curl http://localhost:3001/v1/chat/completions \
  -H "Authorization: Bearer freellmapi-votre-cle-unifiee" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "auto",
    "messages": [{"role": "user", "content": "Bonjour !"}]
  }'
```

Chaque réponse porte un en-tête `X-Routed-Via: <platform>/<model>` indiquant quel fournisseur a réellement servi la requête.

## Connexion d'OpenCode

OpenCode accepte n'importe quel endpoint OpenAI-compatible comme provider custom via le paquet `@ai-sdk/openai-compatible` et une entrée dans `opencode.json`. Deux méthodes :

### Option A : commande automatique (recommandée)

La CLI FreeLLMAPI génère et fusionne la config toute seule (sauvegarde de l'existant, catalogue de modèles vivant, clé référencée par variable d'environnement) :

```bash
npx freellmapi setup-opencode --url http://localhost:3001 --api-key freellmapi-votre-cle-unifiee
```

Puis exporter la clé avant de lancer OpenCode :

```bash
export FREELLMAPI_API_KEY=freellmapi-votre-cle-unifiee
opencode
```

La commande écrit `~/.config/opencode/opencode.json` (fusion structurelle : vos providers et réglages existants sont conservés) et pose `freellmapi/auto` comme modèle par défaut. Redémarrer OpenCode après modification de la config.

### Option B : configuration manuelle

Créer ou éditer `~/.config/opencode/opencode.json` (ou `opencode.json` à la racine d'un projet) :

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "freellmapi/auto",
  "provider": {
    "freellmapi": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "FreeLLMAPI (local)",
      "options": {
        "baseURL": "http://localhost:3001/v1",
        "apiKey": "{env:FREELLMAPI_API_KEY}"
      },
      "models": {
        "auto": {
          "name": "FreeLLMAPI Auto Router",
          "limit": { "context": 1048576, "output": 65536 }
        }
      }
    }
  }
}
```

Points d'attention :

- `baseURL` doit inclure le suffixe `/v1` — OpenCode (comme tous les clients OpenAI-compatibles) n'ajoute que le chemin de complétion.
- `apiKey` utilise la substitution `{env:FREELLMAPI_API_KEY}` : la clé reste hors du fichier de config. Exporter `FREELLMAPI_API_KEY` dans le shell (ou `~/.bashrc` / `~/.zshrc`) avant de lancer `opencode`.
- Le modèle `auto` doit être déclaré dans la map `models` : OpenCode ne résout une référence `provider/model` que si l'id du modèle y figure.
- Sans `"model": "freellmapi/auto"` en haut du fichier, OpenCode garde le modèle précédemment sélectionné — le définir explicitement garantit le démarrage sur le routeur.

Vérification dans OpenCode :

```text
/models        # la liste doit montrer freellmapi/auto et le catalogue
```

Puis lancer une session normalement : `opencode`.

### Lister les modèles disponibles

```bash
curl http://localhost:3001/v1/models \
  -H "Authorization: Bearer freellmapi-votre-cle-unifiee"
```

Pour épingler un modèle précis au lieu du routeur auto, récupérez son id dans cette liste et remplacez `auto` dans la config (ex. `"model": "freellmapi/<id>"` et ajoutez l'entrée correspondante dans `models`). Les stratégies nommées `auto:fast`, `auto:smart` et les profils de chaîne (`auto:<profil>`) sont aussi utilisables comme valeur de `model` dans les requêtes API.

## Pièges connus

- **`FREELLMAPI_API_KEY` non exportée** : OpenCode démarre mais les requêtes échouent en 401. Vérifier avec `echo $FREELLMAPI_API_KEY` dans le même shell que `opencode`.
- **Changements de config non pris en compte** : OpenCode ne recharge pas `opencode.json` à chaud — redémarrer l'application après chaque édition.
- **`baseURL` sans `/v1`** : les appels échouent en 404. Toujours terminer par `/v1`.
- **Modèle `auto` non déclaré** : la référence `freellmapi/auto` n'est pas résoluble — déclarer `auto` dans la map `models` du provider.
- **Aucune clé fournisseur valide** : le routeur répond mais chaque requête échoue après épuisement de la chaîne de fallback. Vérifier le statut des clés (point de couleur) sur la page Keys du tableau de bord.
- **Le conteneur n'est pas démarré** : `curl http://localhost:3001/v1/models` doit répondre avant tout test OpenCode (`docker compose logs -f freellmapi` en cas de doute).
- **Premium vs gratuit** : la version gratuite suit un snapshot mensuel du catalogue (environ 30 jours de retard sur les nouveaux modèles) ; rien n'est bloqué, les modèles arrivent juste plus tard.

## Références

- [Dépôt GitHub FreeLLMAPI](https://github.com/tashfeenahmed/freellmapi)
- [Guide d'installation et déploiement](https://github.com/tashfeenahmed/freellmapi/blob/main/docs/en/install/01-install.md)
- [Référence API REST](https://github.com/tashfeenahmed/freellmapi/blob/main/docs/en/api/01-rest-api.md)
- [Clients et agents de code (dont OpenCode)](https://github.com/tashfeenahmed/freellmapi/blob/main/docs/en/clients/01-agent-clients.md)
- [Catalogue des modèles gratuits](https://freellmapi.co/models.html)
- [Documentation OpenCode — providers](https://opencode.ai/docs/providers/)
- [Schéma de configuration OpenCode](https://opencode.ai/config.json)

---

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
| 1 | [Fonctionnement](articles/llm-fondamentaux/01-fonctionnement.md) | Expliquer la prédiction de tokens, l'attention et l'inférence à un collègue |
| 2 | [Entraînement](articles/llm-fondamentaux/02-entrainement.md) | Situer le pré-entraînement, le SFT et l'alignement RLHF dans le cycle de vie d'un modèle |
| 3 | [Hallucinations](articles/llm-fondamentaux/03-hallucinations.md) | Définir une hallucination, la repérer et réduire le risque |
| 4 | [Température et sampling](articles/llm-fondamentaux/04-temperature.md) | Régler température, top-p et top-k selon le cas d'usage |
| 5 | [Prompt engineering](articles/llm-fondamentaux/05-prompt-engineering.md) | Rédiger un prompt structuré (rôle, contexte, instruction, format) et itérer |
| 6 | [Fine-tuning et RAG](articles/llm-fondamentaux/06-fine-tuning-rag.md) | Choisir entre fine-tuning, RAG et simple prompting selon le besoin |
| 7 | [Créer un agent IA](articles/llm-fondamentaux/07-agent-ia.md) | Construire une boucle agent (raisonnement + outils) avec MCP et garde-fous |
| 8 | [Bilan des modèles](articles/llm-fondamentaux/08-bilan-modeles.md) | Choisir un modèle (fermé ou open weights) selon coût, latence, confidentialité et usage |
| 9 | [Réseaux de neurones](articles/llm-fondamentaux/09-reseaux-de-neurones.md) | Expliquer neurone, couches, propagation avant, rétropropagation et le passage au transformer |
| 10 | [Inférence](articles/llm-fondamentaux/10-inference.md) | Comprendre le déroulement de l'inférence : cache KV, quantification, batching, décodage spéculatif, moteurs de serving |

## Ordre de lecture recommandé

1. Chapitres 1, 2 et 9 pour le vocabulaire commun (le chapitre 9 pose les fondations théoriques : réseaux de neurones et entraînement).
2. Chapitre 10 pour comprendre ce qui se passe concrètement quand on appelle un modèle (inférence).
3. Chapitres 3 et 4 pour comprendre les deux comportements qui surprennent toujours : les hallucinations et le hasard contrôlé.
4. Chapitre 5 avant tout projet concret : c'est le levier le plus rapide et le moins coûteux.
5. Chapitre 6 quand le prompting ne suffit plus.
6. Chapitre 7 quand le modèle doit agir, pas seulement répondre.
7. Chapitre 8 pour le choix du modèle, à relire régulièrement : ce chapitre date vite.

## Export PDF

À chaque pull request modifiant ce répertoire, le workflow [`pdf-llm-fondamentaux`](.github/workflows/pdf-llm-fondamentaux.yml) concatène les chapitres 1 à 10 dans l'ordre numérique et génère un PDF (`llm-fondamentaux.pdf`), publié en artefact du run et signalé en commentaire de la PR.

## Pièges connus

- **Les chapitres 1 à 7 et 9-10 sont stables** : les concepts évoluent lentement. Le chapitre 8, lui, reflète l'état du marché en octobre 2026 et doit être relu tous les 2-3 mois.
- **Ne pas confondre « poids ouverts » (open weights) et « open source »** : la plupart des modèles « ouverts » publient leurs poids sous des licences restrictives (voir chapitre 8).
- **Un LLM ne « sait » rien** : il génère du texte plausible. Toute sortie factuelle mérite vérification (chapitre 3).

## Références

- [Hugging Face — documentation LLM](https://huggingface.co/docs/transformers/llm_tutorial) — tutoriels pratiques sur les transformers
- [Lilian Weng — « Large Language Model » (blog)](https://lilianweng.github.io/posts/2023-01-27-the-transformer-family/) — synthèse technique de référence
- [Artificial Analysis — LLM Leaderboard](https://artificialanalysis.ai/leaderboards/models) — classement indépendant des modèles (intelligence, vitesse, prix)
- [3Blue1Brown — « But what is a GPT? » (vidéo)](https://www.3blue1brown.com/lessons/gpt) — visualisation de l'architecture transformer

---

# Chapitre 1 — Fonctionnement d'un LLM

> Partie de : [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md)

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

Le mécanisme central du transformer s'appelle **l'attention** (self-attention). (Les fondations — neurone, couches, propagation avant, rétropropagation — sont détaillées au chapitre 9.) Pour chaque token, le modèle calcule à quels tokens précédents il doit « prêter attention ».

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
| Inférence | « Serving » | Les poids sont figés. Le modèle génère du texte, token par token, sur votre requête. C'est ce que vous payez quand vous appelez une API (voir chapitre 10 pour le déroulement détaillé de l'inférence). |

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

Faire tourner un LLM en production demande des optimisations, sinon le coût explose (développé au chapitre 10) :

- **Quantification** : les poids sont stockés en précision réduite (FP16, INT8, INT4, voire 1-2 bits). Un modèle de 70 milliards de paramètres tient ainsi sur 1-2 GPU au lieu de 4-8. Léger sacrifice de qualité, énorme gain de coût.
- **Cache KV** : les étapes d'attention pour les tokens déjà traités sont mises en cache, évitant de les recalculer à chaque nouveau token.
- **Batching continu** et **spéculative decoding** (un petit modèle « brouillon » propose plusieurs tokens, le grand modèle les vérifie en un seul passage) : accélèrent la génération.
- **Serving local** : des moteurs comme llama.cpp, Ollama, vLLM ou LM Studio font tourner des modèles sur votre propre machine (voir l'article [LM Studio sur Ubuntu](articles/lm-studio/README.md)).

## Pièges connus

- **« Le modèle a été entraîné sur X donc il sait X »** : faux. L'entraînement crée des patterns, pas une base de connaissances fiable (voir chapitre 3 sur les hallucinations).
- **Confondre taille et qualité** : un modèle plus gros n'est pas automatiquement meilleur pour votre tâche ; un petit modèle bien choisi (ou fine-tuné) peut suffiser.
- **Sous-estimer le coût au token** : un prompt système de 5 000 tokens envoyé à chaque requête coûte 5 000 tokens à chaque appel, même si la réponse fait 20 tokens.

## Références

- [3Blue1Brown — « But what is a GPT? »](https://www.3blue1brown.com/lessons/gpt) — visualisation animée de l'architecture (en anglais, accessible)
- [Lilian Weng — « The Transformer Family »](https://lilianweng.github.io/posts/2023-01-27-the-transformer-family/) — synthèse technique complète (en anglais)
- [Hugging Face — « LLM Tutorial »](https://huggingface.co/docs/transformers/llm_tutorial) — mise en pratique avec des modèles open weights
- [Andrej Karpathy — « Intro to Large Language Models » (vidéo, 1h)](https://www.youtube.com/watch?v=zjkBMFhNj_g) — conférence de référence (en anglais)
- [colibri — moteur d'inférence MoE en C pur](articles/colibri/README.md) — exemple concret d'optimisations d'inférence (VRAM/RAM/disque, cache KV, MTP)

---

# Chapitre 2 — L'entraînement d'un LLM

> Partie de : [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md)

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

---

# Chapitre 3 — Les hallucinations

> Partie de : [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md)

**Date :** 2026-10-29

## Objectif

Définir précisément ce qu'est une hallucination, comprendre pourquoi les LLM en produisent, et appliquer les techniques qui réduisent le risque dans une application réelle.

## Définition

Une **hallucination** est une sortie d'un LLM qui :

- est **fluente et convaincante**, mais
- est **factuellement fausse, inventée ou non fondée** sur le contexte fourni.

Exemples typiques :

- Citer un article scientifique, un arrêt de jurisprudence ou un livre **qui n'existe pas**.
- Donner une date, un chiffre ou une statistique **inventée**.
- Affirmer avec assurance quelque chose de **faux** (ex. : « Paris est la capitale de l'Espagne »).
- Résumer un document en y ajoutant des **détails absents** du document.

**Point clé** : le modèle ne « sait » pas qu'il hallucine. Il génère le texte le plus *plausible*, pas le plus *vrai*. La fluidité n'est pas un indicateur de fiabilité.

## Pourquoi les LLM hallucinent

Trois causes structurelles (voir chapitres 1 et 2) :

1. **L'objectif d'entraînement est la plausibilité, pas la vérité.** Le modèle apprend « quel token suit le plus probablement », pas « quel fait est vérifié ». Un nom d'auteur inventé mais crédible a une probabilité élevée.
2. **La connaissance est compressée et imparfaite.** Les faits sont stockés sous forme de patterns statistiques, pas de base de connaissances. Les détails rares sont les premiers à être « oubliés » ou reconstruits de travers.
3. **La date de coupure.** Le modèle ne connaît rien de ce qui est arrivé après son corpus d'entraînement. Toute question sur l'actualité déclenche un risque d'invention.

Un quatrième facteur, plus subtil : **l'alignement pousse à répondre**. Un modèle aligné est entraîné à être utile et à ne pas refuser. Face à une question à laquelle il ne « sait » pas répondre, il préfère souvent inventer une réponse plausible plutôt que de dire « je ne sais pas » — surtout si le prompt ne l'y autorise pas explicitement.

## Les différents types d'hallucinations

| Type | Description | Exemple |
|------|-------------|---------|
| **Intrinsèque** | Contradit le contexte fourni dans le prompt | Le document dit « contrat signé en 2023 », le résumé indique « 2024 » |
| **Extrinsèque** | Inventé de toutes pièces, invérifiable dans le contexte | Une fausse référence bibliographique |
| **Fausses attributions** | Action ou parole attribuée à la mauvaise personne | « Selon Einstein, … » pour une citation apocryphe |
| **Chiffres inventés** | Statistiques, montants, pourcentages sans source | « 73 % des entreprises » sans étude derrière |
| **Confabulation conversationnelle** | Le modèle « oublie » ce qu'il a dit plus tôt et se contredit | Deux réponses successives incompatibles dans un même chat |

## Comment réduire les hallucinations

### 1. Autoriser explicitement l'abstention

La mesure la plus efficace et la moins coûteuse. Dans le prompt système :

```
Si tu n'es pas sûr d'un fait, ou si l'information n'est pas dans le
contexte fourni, réponds « Je ne sais pas » plutôt que d'inventer.
Ne cite jamais une source que tu ne peux pas vérifier.
```

### 2. Ancrer sur un contexte fourni (RAG)

Fournir les documents pertinents dans le prompt et interdire de sortir de ce cadre :

```
Réponds UNIQUEMENT à partir des documents ci-dessous. Si la réponse
n'y figure pas, dis-le.

<documents>
{{DOCUMENTS}}
</documents>

Question : {{QUESTION}}
```

C'est le principe du **RAG** (Retrieval-Augmented Generation, voir chapitre 6). Le taux d'hallucination chute drastiquement quand le modèle est contraint à un corpus fermé.

### 3. Demander des citations vérifiables

```
Pour chaque affirmation factuelle, cite le passage exact du document
source entre guillemets, avec le nom du document.
```

Puis **vérifier mécaniquement** : le passage cité existe-t-il vraiment dans le document ? (Un simple test de chaîne de caractères élimine beaucoup d'hallucinations de citation.)

### 4. Baisser la température

Une température basse (0 à 0,3) réduit la créativité — donc l'invention (voir chapitre 4). Pour toute tâche factuelle, rester bas.

### 5. Décomposer la tâche

Demander au modèle de **raisonner étape par étape** (« réfléchis avant de répondre ») améliore la fiabilité sur les tâches complexes. Mieux : séparer **recherche** et **rédaction** en deux appels distincts.

### 6. La vérification externe (grounding)

- **Recherche web** : connecter le modèle à un moteur de recherche pour qu'il ancre ses réponses (tool use, chapitre 7).
- **Vérification croisée** : faire relire la réponse par un second modèle (ou un second passage) en lui demandant de signaler les affirmations non sourcées.
- **Déterminisme sur les faits** : pour les données structurées (dates, montants, codes), ne jamais laisser le LLM les générer librement — les injecter depuis une base de données.

### 7. Le fine-tuning ciblé

Fine-tuner sur des exemples du domaine réduit les hallucinations **dans ce domaine** (chapitre 6). À réserver aux cas où le prompting + RAG ne suffisent pas.

## Évaluer le taux d'hallucination

On ne peut pas « faire confiance » sans mesurer. Méthodes concrètes :

- **Jeu de test golden set** : un ensemble de 50-200 questions dont vous connaissez la bonne réponse (ou l'absence de réponse). Mesurer le taux de réponses correctes, fausses, et « je ne sais pas ».
- **Vérification humaine** : échantillonner les sorties et les faire noter (coûteux, réservé à la validation).
- **Vérification automatique** : pour les citations, vérifier que les passages existent ; pour les entités (noms, dates), les comparer à une base de référence.

**Métrique utile** : taux d'hallucination = nombre d'affirmations factuelles fausses / nombre total d'affirmations factuelles. Un bon système RAG descend sous les 2-5 % sur son domaine ; un LLM brut peut dépasser 20-30 % sur les questions factuelles pointues.

## Hallucination ou pas ? Cas limites

- **Résumé fidèle mais incomplet** : ce n'est pas une hallucination, c'est un biais de sélection. Le résumeur choisit quoi garder.
- **Interprétation raisonnable d'un texte ambigu** : le LLM « prend position » là où un humain hésiterait. Formuler la question plus précisément.
- **Réponse obsolète** : le fait était vrai à la date d'entraînement, faux aujourd'hui. Ce n'est pas une hallucination au sens strict, mais le risque est identique — d'où l'intérêt de la recherche web en temps réel.

## Pièges connus


- **« Le modèle a cité une source, donc c'est vrai »** : les citations sont le domaine d'hallucination le plus fréquent. Toujours vérifier.
- **La confiance affichée trompe** : un modèle hésitant et un modèle halluciné peuvent paraître aussi assurés l'un que l'autre. Ne jamais utiliser le ton comme signal de fiabilité.
- **Les petits modèles hallucinent plus** : à tâche égale, un modèle de 3B inventera plus qu'un frontier. Ne pas lésiner sur la taille pour les tâches factuelles critiques.
- **Le RAG n'élimine pas le risque** : il le réduit fortement, mais le modèle peut encore mal interpréter les documents ou « compléter » au-delà.

## Références

- [Hugging Face — « LLM Hallucinations » (blog)](https://huggingface.co/blog/llm-hallucinations) — panorama des causes et solutions (en anglais)
- [Vectara — « Hallucination Leaderboard »](https://github.com/vectara/hallucination-leaderboard) — classement public des modèles sur la fidélité au résumé (en anglais)
- [Anthropic — « Claude's system prompt » (exemples)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — techniques d'ancrage (en anglais)
- [Stanford CRFM — HELM](https://crfm.stanford.edu/helm/latest/) — benchmarks incluant la fiabilité factuelle (en anglais)
- [Article académique — « A Survey on Hallucination in LLMs » (arXiv)](https://arxiv.org/abs/2311.05232) — revue complète (en anglais)

---

# Chapitre 4 — Température et paramètres de génération

> Partie de : [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md)

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

---

# Chapitre 5 — Le prompt engineering

> Partie de : [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md)

**Date :** 2026-10-29

## Objectif

Savoir rédiger un prompt structuré et efficace — rôle, contexte, instruction, format de sortie — appliquer les techniques éprouvées (exemples, chaîne de pensée, balises) et mettre en place une démarche d'itération mesurée.

## Définition

Le **prompt engineering** est l'art de formuler les instructions envoyées à un LLM pour obtenir le résultat voulu, **sans modifier les poids du modèle**. C'est le levier le plus rapide, le moins cher et le plus réversible : on change du texte, on reteste, on garde ce qui marche.

Ordre de priorité des leviers (du plus simple au plus lourd) :

```
1. Prompt engineering   →  heures, gratuit, réversible
2. RAG (chapitre 6)     →  jours, coût infra, réversible
3. Fine-tuning (chapitre 6) → jours/semaines, coût GPU, nouvelle version du modèle
```

**Toujours commencer par le prompt engineering.** Passer au RAG ou au fine-tuning avant d'avoir un prompt solide gaspille du temps et de l'argent.

## Anatomie d'un bon prompt

Un prompt efficace suit une structure en quatre blocs. Même les modèles récents, très bons pour comprendre le langage naturel, bénéficient énormément d'une structure explicite.

```
┌─────────────────────────────────────────────┐
│  RÔLE (system prompt)                       │  Qui est le modèle, dans quel ton
│  « Tu es un réviseur de code senior… »       │
├─────────────────────────────────────────────┤
│  CONTEXTE                                   │  Pourquoi la tâche existe, pour qui,
│  « L'utilisateur est développeur junior… »   │  quelles sont les contraintes
├─────────────────────────────────────────────┤
│  INSTRUCTION (tâche)                        │  Verbe d'action + périmètre précis
│  « Relis ce code et signale les bugs… »      │
├─────────────────────────────────────────────┤
│  FORMAT DE SORTIE                           │  Structure exacte attendue
│  « Réponds en JSON : {bugs: [...], …} »     │
└─────────────────────────────────────────────┘
        +  EXEMPLES (few-shot) en annexe si besoin
        +  DONNÉES (le contenu à traiter)
```

### Exemple complet

```markdown
Tu es un assistant spécialisé dans la rédaction de notes de frais
professionnelles, pour des commerciaux en déplacement.

RÈGLES :
- Ton professionnel mais concis (2-3 phrases maximum par champ).
- Si une information manque, écris « À COMPLÉTER » plutôt que d'inventer.
- Ne jamais inventer de montants, de dates ou de noms de clients.

TÂCHE :
À partir de la note de frais brute ci-dessous, produis un récapitulatif
structuré.

FORMAT DE SORTIE (JSON strict, rien d'autre) :
{
  "destination": "ville",
  "date_debut": "AAAA-MM-JJ",
  "date_fin": "AAAA-MM-JJ",
  "motif": "1 phrase",
  "depenses": [{"type": "transport|hotel|restauration", "montant_estime": "à défaut, null"}],
  "points_a_clarifier": ["..."]
}

NOTE DE FRAIS BRUTE :
<note>
Départ mardi pour Lyon, retour jeudi. Train AR 120 €, hôtel 2 nuits
x 95 €, 3 restos avec clients. Le client Durand a annulé mercredi,
j'ai quand même pris l'hôtel.
</note>
```

## Les règles d'or

1. **Être spécifique.** « Sois clair » ne veut rien dire. « Écris des phrases de moins de 20 mots, vocabulaire de niveau lycée » est actionnable.
2. **Montrer le format de sortie.** « Réponds en JSON » est faible ; un exemple JSON concret ou un schéma est fort. Quand l'API le permet, utiliser `response_format` (chapitre 4) pour le garantir.
3. **Délimiter les responsabilités.** « Ne réponds que sur le document fourni », « Si tu ne sais pas, dis-le » (voir chapitre 3).
4. **Un test humain simple** : montrez votre prompt à un collègue qui ne connaît pas le sujet. S'il ne peut pas exécuter la tâche sans vous poser de questions, le modèle non plus.
5. **Le contexte bat la longueur.** Un prompt de 150 mots bien ciblés bat un pavé de 2 000 mots confus. Chaque token coûte de l'argent et de la latence.

## Les techniques principales

### Le rôle (system prompt)

Définir un rôle concentre le comportement du modèle sur un domaine.

```
Tu es un ingénieur senior en revue de code, spécialisé Python.
Tu privilégies la lisibilité et la sûreté sur la concision.
```

Utile pour le ton et les priorités. À ne pas surcharger : un rôle ne remplace pas une instruction précise.

### Le few-shot (exemples dans le prompt)

Montrer 1 à 5 exemples du format attendu. C'est souvent la technique la plus efficace pour le formatage.

```
Classe le sentiment de chaque avis (positif / négatif / neutre).

Avis : « Livraison rapide, produit conforme. » → positif
Avis : « Colis arrivé ouvert, service client injoignable. » → négatif
Avis : « Reçu dans les délais. » → neutre
Avis : « {{AVIS}} » →
```

- **0 exemple** = zero-shot, **1-5** = few-shot. Au-delà, les gains diminuent et le coût en tokens grimpe.
- Choisir des exemples **représentatifs des cas difficiles**, pas seulement les faciles.

### La chaîne de pensée (Chain-of-Thought)

Demander au modèle de raisonner étape par étape avant de conclure. Sur les tâches de raisonnement (maths, logique, analyse), le gain est significatif.

```
Analyse ce contrat étape par étape :
1. Identifie les clauses inhabituelles.
2. Pour chacune, explique le risque.
3. Conclus par une recommandation (accepter / négocier / refuser).
```

**Variante structurée** : demander d'abord l'analyse, puis la conclusion dans un format séparé. Les modèles récents de raisonnement (« extended thinking ») intègrent ce mécanisme nativement — activer leur mode réflexion quand la tâche est complexe.

### Les délimiteurs et balises

Quand le prompt mélange plusieurs types de contenu (instructions, contexte, données), les séparer visuellement réduit les confusions :

```markdown
<instructions>
Résume le document ci-dessous en 3 points.
</instructions>

<document>
{{CONTENU_DU_DOCUMENT}}
</document>

Règles : ne cite que le document. Si un point n'y figure pas, omets-le.
```

Conventions utiles : balises XML simples (`<document>`, `<exemple>`), ou séparateurs (`###`, `---`). L'important est la **cohérence** : les mêmes balises partout, dans tous les prompts de l'application.

### Décomposer les tâches (prompt chaining)

Une tâche complexe en une seule étape donne un résultat moyen. La découper en étapes, chacune avec son propre prompt, est plus fiable :

```
Étape 1 : extraire les entités (dates, montants, noms) du texte brut
Étape 2 : valider les entités extraites contre des règles métier
Étape 3 : rédiger le résumé à partir des entités validées
```

Chaque étape est plus simple, donc plus fiable, et chaque sortie intermédiaire est vérifiable.

### Formuler en positif

Dire ce qu'il faut faire plutôt que ce qu'il ne faut pas faire :

- ❌ « Ne sois pas verbeux. » → le modèle entend « verbeux ».
- ✅ « Réponds en 2 phrases maximum. »

## Le prompt système vs le prompt utilisateur

| | System prompt | Message utilisateur |
|---|---|---|
| Rôle | Définit le comportement général, le rôle, les règles permanentes | Porte la tâche concrète et les données |
| Persistance | Envoyé à **chaque** appel de la conversation | Change à chaque tour |
| Coût | Payé à chaque requête (attention aux prompts système trop longs) | Payé une fois par message |
| Bonne pratique | Rôle + règles + format global + garde-fous | Tâche + données + contexte spécifique au tour |

**Piège** : un system prompt trop long et trop chargé « noie » les instructions. Garder le system prompt focalisé ; mettre les détails de la tâche dans le message utilisateur.

## Itérer et évaluer : la méthode

Le prompt engineering est un **processus expérimental**, pas une inspiration.

1. **Partir d'une version simple** (rôle + tâche + format).
2. **Créer un jeu de test** : 20-50 entrées représentatives, avec les sorties attendues (même partielles).
3. **Tester, noter les échecs** : catégoriser (format faux, contenu faux, ton faux, hallucination…).
4. **Modifier UN élément à la fois** : changer deux choses et vous ne saurez pas laquelle a marché.
5. **Re-tester sur le même jeu.** Revenir en arrière si ça régresse.
6. **Versionner les prompts** : les stocker dans des fichiers (YAML/Markdown), pas dans le code en dur. Un prompt est un artefact de production.

### Grille d'évaluation rapide (sans vérité terrain complète)

| Critère | Comment évaluer |
|---------|-----------------|
| Fidélité à la tâche | La sortie fait-elle ce qui était demandé ? |
| Format | Structure respectée ? (testable automatiquement pour du JSON) |
| Factualité | Les affirmations sont-elles sourcées / vérifiables ? |
| Hallucinations | Le modèle a-t-il inventé ? (voir chapitre 3) |
| Ton / style | Conforme au rôle demandé ? |
| Coût | Nombre de tokens entrée + sortie par requête |

### Exemple de fiche de test

```yaml
# tests/resume-contrat.yaml
- id: resume-01
  entree: "<texte de contrat court>"
  attendu:
    nb_points_max: 3
    contient_date_signature: true
  a_verifier_humain: true
```

## Les pièges classiques

- **Le « prompt magique » copié-collé** : un prompt trouvé sur Internet est optimisé pour un modèle et un tâche précis. Toujours l'adapter et le re-tester.
- **Sur-instruire** : 30 règles contradictoires produisent un modèle perdu. Prioriser ; déplacer les règles secondaires dans des exemples plutôt que dans des interdictions.
- **Négliger les cas limites** : un prompt qui marche sur les exemples « propres » échoue en production sur les entrées bizarres (texte vide, langue étrangère, injection). Tester les cas limites explicitement.
- **L'injection de prompt** : un utilisateur malveillant peut inclure dans ses données des instructions du type « ignore les instructions précédentes ». Défenses : séparer clairement instructions et données (balises), ne jamais faire confiance au contenu utilisateur comme s'il était une instruction (voir chapitre 7 sur les agents).
- **Oublier de re-tester après un changement de modèle** : un prompt optimisé pour un modèle n'est pas optimal pour un autre.

## Quand le prompt engineering ne suffit plus

Signaux qu'il faut passer à la vitesse supérieure (chapitre 6) :

- Le modèle ne parvient pas à adopter un **style ou un format très spécifique**, même avec beaucoup d'exemples.
- Il lui manque des **connaissances métier pointues** que le contexte ne peut pas fournir à chaque requête.
- Le taux d'hallucination reste élevé malgré RAG et température basse.
- Le coût du few-shot (dizaines d'exemples à chaque requête) devient prohibitif.

## Références

- [Anthropic — « Prompt engineering overview »](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — guide officiel, techniques à jour (en anglais)
- [Anthropic — « Prompt engineering best practices »](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/claude-prompting-best-practices) — balises, rôle, exemples (en anglais)
- [Prompting Guide — promptingguide.ai](https://www.promptingguide.ai/) — ressource communautaire complète (en anglais)
- [OpenAI — « Prompt engineering »](https://platform.openai.com/docs/guides/prompt-engineering) — stratégies officielles (en anglais)
- [Anthropic — tutoriel interactif de prompt engineering (GitHub)](https://github.com/anthropics/prompt-eng-interactive-tutorial) — exercices guidés (en anglais)
- [Lilian Weng — « Prompt Engineering »](https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/) — synthèse académique (en anglais)

---

# Chapitre 6 — Fine-tuning et RAG

> Partie de : [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md)

**Date :** 2026-10-29

## Objectif

Savoir choisir entre **prompt engineering**, **RAG** et **fine-tuning** selon le besoin, comprendre le fonctionnement de chacun, et disposer de recettes concrètes pour mettre en œuvre le RAG et un fine-tuning léger (LoRA).

## Le décisionnaire : quelle approche pour quel besoin

Trois leviers, trois périmètres. La question n'est pas « lequel est le meilleur » mais « lequel résout mon problème au moindre coût ».

| Besoin | Solution | Pourquoi |
|--------|----------|----------|
| Mieux formuler la tâche, structurer la sortie | **Prompt engineering** (chapitre 5) | Gratuit, immédiat, réversible |
| Le modèle doit connaître **vos documents / données métier** | **RAG** | Les connaissances sont injectées à la requête, sans réentraîner |
| Le modèle doit adopter un **style, un format, un comportement** très spécifique | **Fine-tuning** (souvent LoRA) | Le comportement est appris dans les poids |
| Le modèle doit raisonner mieux sur **votre domaine** (juridique, médical, code interne) | **Fine-tuning** + RAG combinés | Les deux : connaissances (RAG) + comportement (fine-tuning) |
| Réduire le coût en tokens | Prompt engineering + RAG ciblé | Moins de contexte = moins de tokens |
| Données sensibles, pas de fuite vers un fournisseur | Modèle local + RAG local | Voir chapitre 8 et l'article [LM Studio](articles/lm-studio/README.md) |

**Règle d'or** : essayer dans l'ordre prompt → RAG → fine-tuning. Chaque étape coûte plus cher que la précédente en temps, en infrastructure et en maintenance.

## Le RAG (Retrieval-Augmented Generation)

### Principe

Le RAG consiste à **rechercher les passages pertinents** dans une base de documents, puis à les **injecter dans le prompt** du LLM, qui répond en s'ancrant sur ce contexte.

```
Question de l'utilisateur
        │
        ▼
┌─────────────────┐     ┌────────────────────────┐     ┌─────────────┐
│  1. Recherche   │────►│  2. Construction        │────►│  3. LLM     │
│  (retrieval)    │     │  du prompt              │     │  (génération│
│                 │     │                         │     │   ancrée)   │
│  « Quels morceaux│     │  Question + passages    │     │             │
│   parlent de X ?»│     │  pertinents + règles    │     │             │
└─────────────────┘     └────────────────────────┘     └─────────────┘
        ▲
        │
   Base documentaire indexée
   (vector store + éventuellement recherche hybride)
```

Le modèle ne « sait » toujours rien de plus qu'avant : on lui **donne** l'information au moment de la requête. C'est la différence fondamentale avec le fine-tuning.

### Pourquoi le RAG réduit les hallucinations

Le prompt contient une instruction du type : « Réponds UNIQUEMENT à partir des documents ci-dessous. Si l'information n'y figure pas, dis-le. » Le modèle est contraint à un corpus fermé et vérifiable (voir chapitre 3).

### Les composants d'un RAG

1. **Le découpage (chunking)** : les documents sont découpés en morceaux de ~200-800 tokens, avec un léger chevauchement (10-20 %) pour ne pas couper une idée en deux.
2. **L'embedding** : chaque morceau est transformé en vecteur dense (un modèle d'embedding, ex. : `text-embedding-3`, BGE, E5, ou un modèle local via Ollama/Sentence Transformers).
3. **L'indexation** : les vecteurs sont stockés dans une **base vectorielle** (Chroma, FAISS, Qdrant, pgvector, Pinecone, Weaviate…).
4. **La recherche** : à la requête, la question est embarquée ; on récupère les k morceaux les plus proches (similarité cosinus). k = 3 à 10 typiquement.
5. **L'augmentation du prompt** : les morceaux sont insérés dans le prompt avec la question.
6. **La génération** : le LLM répond, idéalement avec des citations des sources.

### Les variantes qui améliorent la qualité

| Technique | Principe | Quand l'utiliser |
|-----------|----------|------------------|
| **Recherche hybride** | Combiner similarité vectorielle + recherche par mots-clés (BM25) | Vocabulaire métier précis, acronymes, codes |
| **Re-ranking** | Un modèle de re-classement (cross-encoder) réordonne les candidats | Précision critique, k élevé (20-50) en entrée |
| **Requête reformulée** | Le LLM reformule la question en 2-3 variantes pour élargir la recherche | Questions ambiguës ou mal formulées |
| **RAG multi-hop** | Plusieurs tours de recherche enchaînés | La réponse dépend de plusieurs documents liés |
| **RAG avec vérification** | Un second passage vérifie que la réponse est bien fondée sur les sources | Domaine à risque (juridique, médical) |
| **Graphe de connaissances (GraphRAG)** | Indexer les relations entre entités, pas seulement les morceaux | Données fortement relationnelles |

### Exemple minimal de RAG en Python

```python
# Prérequis : pip install openai chromadb sentence-transformers
# Le vector store et l'embedding tournent en local ; seul le LLM est appelé en API.
import chromadb
from sentence_transformers import SentenceTransformer
from openai import OpenAI

embedder = SentenceTransformer("BAAI/bge-m3")          # modèle d'embedding local
client_llm = OpenAI()                                  # clé API LLM

chroma = chromadb.PersistentClient(path="./chroma_db")
collection = chroma.get_or_create_collection("docs")

def indexer(documents: list[str]) -> None:
    collection.add(
        documents=documents,
        embeddings=embedder.encode(documents).tolist(),
        ids=[f"doc-{i}" for i in range(len(documents))],
    )

def repondre(question: str, k: int = 5) -> str:
    resultats = collection.query(
        query_embeddings=embedder.encode([question]).tolist(),
        n_results=k,
    )
    contexte = "\n\n---\n\n".join(resultats["documents"][0])

    reponse = client_llm.chat.completions.create(
        model="gpt-5-mini",
        temperature=0,
        messages=[
            {"role": "system", "content": (
                "Réponds UNIQUEMENT à partir des documents fournis. "
                "Si la réponse n'y figure pas, réponds « Je ne sais pas ». "
                "Cite tes sources entre crochets [doc n]."
            )},
            {"role": "user", "content": f"Documents :\n{contexte}\n\nQuestion : {question}"},
        ],
    )
    return reponse.choices[0].message.content
```

### Pièges du RAG

- **« Garbage in, garbage out » sur les documents** : un RAG sur des documents obsolètes ou mal structurés donne des réponses fausses *avec l'air de sources fiables*.
- **Le mauvais morceau** : si la recherche récupère le mauvais passage, le modèle répond avec assurance sur le mauvais sujet.
- **Le contexte trop long** : 10 morceaux x 800 tokens = 8 000 tokens de contexte à chaque requête. Coût et latence montent vite. D'où l'intérêt du re-ranking et d'un k raisonnable.
- **Les tableaux, images et mises en page** : le découpage naïf détruit les tableaux et les schémas. Utiliser des parseurs adaptés (PDF → markdown structuré, tableaux en CSV/JSON).

## Le fine-tuning

### Principe

Le **fine-tuning** continue l'entraînement du modèle (chapitre 2, phases 2-4) **sur vos propres données**, pour modifier son comportement de façon persistante.

- **Fine-tuning complet** : tous les poids sont ajustés. Puissant, mais coûteux (il faut charger le modèle entier en mémoire GPU) et risqué (oubli catastrophique : le modèle oublie ce qu'il savait).
- **PEFT / LoRA (Low-Rank Adaptation)** : on fige le modèle et on n'entraîne qu'une petite matrice de rang faible ajoutée à chaque couche. **~1 % des paramètres** ajustés, résultats souvent équivalents pour l'adaptation de style/format. C'est la méthode standard en 2026.

### Quand fine-tuner plutôt que faire du RAG

| Fine-tuning | RAG |
|-------------|-----|
| Apprendre un **comportement** (style, ton, format, réflexes) | Fournir des **connaissances** (faits, documents) |
| Le savoir-faire doit être **persistant** dans le modèle | Les connaissances **changent souvent** (il suffit de réindexer) |
| Coût fixe à l'entraînement, coût d'inférence inchangé | Coût au token (contexte plus long), maintenance de l'index |
| Nouvelle version du modèle à déployer et à maintenir | Mise à jour des documents sans réentraînement |

**La plupart des projets métiers ont besoin des deux** : RAG pour les connaissances à jour, LoRA pour le style et les réflexes métier.

### Recette LoRA avec les outils standards

```bash
# Prérequis : pip install transformers peft datasets accelerate trl torch
# Nécessite un GPU avec suffisamment de VRAM (ou du QLoRA, voir ci-dessous).
```

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import LoraConfig
from trl import SFTTrainer
from datasets import load_dataset

model_id = "Qwen/Qwen3-8B"  # modèle de base open weights

tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id, device_map="auto")

lora = LoraConfig(
    r=16,                # rang de la décomposition (8-64 typiquement)
    lora_alpha=32,       # échelle (souvent 2 x r)
    target_modules=["q_proj", "v_proj"],  # couches à adapter
    lora_dropout=0.05,
    task_type="CAUSAL_LM",
)

# dataset au format "conversations" : une colonne avec l'échange complet
dataset = load_dataset("json", data_files="mes_exemples.jsonl")["train"]

trainer = SFTTrainer(
    model=model,
    processing_class=tokenizer,
    train_dataset=dataset,
    args={"output_dir": "./lora-out", "num_train_epochs": 3,
          "per_device_train_batch_size": 4, "learning_rate": 2e-4},
    peft_config=lora,
)
trainer.train()
trainer.save_model("./lora-out")
```

**QLoRA** : version quantifiée du LoRA — le modèle de base est chargé en 4 bits, ce qui permet de fine-tuner un modèle de 8-14B sur un GPU grand public (ex. : RTX 4090, 24 Go), voire de 30-70B sur du matériel plus conséquent.

### Préparer les données de fine-tuning

La qualité des données prime sur la quantité (chapitre 2) :

- **500 à 5 000 exemples** bien construits suffisent pour un LoRA efficace sur un style/format.
- Format : paires (instruction, réponse idéale), couvrant les **cas normaux ET les cas limites** (erreurs fréquentes, refus attendus, format spécial).
- **Dédupliquer**, vérifier l'orthographe, s'assurer que les réponses sont réellement ce que vous voulez.
- Mélanger avec ~10-20 % d'exemples génériques pour ne pas faire oublier au modèle ses capacités de base.

### Héberger et servir un modèle fine-tuné

- **Via API fournisseur** : la plupart (OpenAI, Mistral, Together, Fireworks…) permettent d'uploader un fine-tune. Le plus simple.
- **En local** : merger les poids LoRA avec le modèle de base, puis servir avec vLLM, Ollama ou llama.cpp (voir [LM Studio](articles/lm-studio/README.md) et [colibri](articles/colibri/README.md)).

## Combiner RAG + fine-tuning + prompt engineering

L'architecture cible pour un projet métier sérieux :

```
System prompt (rôle, règles, garde-fous)      ← prompt engineering
        │
        ▼
Contexte RAG (documents métier à jour)        ← RAG
        │
        ▼
Modèle fine-tuné (style, réflexes, format)    ← LoRA
        │
        ▼
Post-traitement (validation, citations)       ← code déterministe
```

Chaque couche a un rôle distinct et réversible. C'est plus maintenable qu'un seul gros système.

## Pièges connus

- **Fine-tuner pour des faits** : le modèle oublie, hallucine toujours, et les faits deviennent obsolètes. Le RAG est fait pour ça.
- **RAG pour du style** : injecter 10 exemples de style dans le prompt coûte cher et marche moyennement. Le LoRA est fait pour ça.
- **Sous-estimer la maintenance du RAG** : réindexer à chaque mise à jour documentaire, surveiller la qualité de recherche, versionner les embeddings.
- **Le fine-tuning qui « dégrade » le modèle** : trop de données bruitées ou un learning rate trop haut = modèle moins bon partout. Toujours évaluer avant/après sur un jeu de test.
- **Négliger l'évaluation** : un fine-tune sans mesure de qualité est un pari. Benchmarker sur 50-200 cas avant de déployer (même méthode que chapitre 5).

## Références

- [Hugging Face — « Fine-tuning » (Transformers)](https://huggingface.co/docs/transformers/training) — guide officiel
- [Hugging Face — PEFT (LoRA/QLoRA)](https://huggingface.co/docs/peft/index) — documentation de la bibliothèque de référence
- [Hugging Face — TRL (SFTTrainer)](https://huggingface.co/docs/trl/index) — entraînement supervisé simplifié
- [LangChain — « RAG »](https://python.langchain.com/docs/tutorials/rag/) — tutoriel RAG pas à pas
- [LlamaIndex — « RAG »](https://docs.llamaindex.ai/en/stable/) — framework RAG (chunking, retrieval, évaluation)
- [Pinecone — « What is RAG? »](https://www.pinecone.io/learn/retrieval-augmented-generation/) — introduction pédagogique (en anglais)
- [Chroma — « RAG guide »](https://www.trychroma.com/blog) — base vectorielle légère, bonne pour démarrer

---

# Chapitre 7 — Créer un agent IA

> Partie de : [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md)

**Date :** 2026-10-29

## Objectif

Comprendre ce qu'est un agent IA (par opposition à un simple chatbot), connaître l'architecture d'une boucle agent, et disposer d'une recette concrète pour construire un premier agent avec outils, mémoire et garde-fous.

## Chatbot ou agent : la différence

| | Chatbot (LLM seul) | Agent IA |
|---|---|---|
| **Entrée** | Texte de l'utilisateur | Texte + objectif + environnement |
| **Capacités** | Générer du texte | **Agir** : appeler des outils, lire/écrire des données, enchaîner des actions |
| **Boucle** | Une requête → une réponse | **Boucle** : percevoir → raisonner → agir → observer → recommencer, jusqu'à l'objectif |
| **Exemples** | ChatGPT en conversation, assistant de rédaction | Assistant qui réserve un voyage, déploie du code, enquête sur un incident |

Un agent, c'est un LLM **au centre d'une boucle**, augmenté d'outils et d'une mémoire. Le LLM décide *quoi faire ensuite* ; les outils exécutent ; le résultat revient dans le contexte ; le LLM décide de la suite.

## L'architecture d'une boucle agent

```
        ┌──────────────────────────────────────────┐
        │                                          ▼
 Objectif + mémoire + historique ──►  LLM (raisonnement)
                                            │
                              ┌─────────────┴──────────────┐
                              ▼                            ▼
                        « Répondre »                  « Appeler un outil »
                        (fin de boucle)              (ex. : recherche web,
                              │                        lire un fichier,
                              ▼                        exécuter du code)
                        Réponse à                 ┌─────────────────┐
                        l'utilisateur             │  Exécution de   │
                                                  │  l'outil        │
                                                  └────────┬────────┘
                                                           │
                                                  Résultat observé
                                                  (retour dans le contexte)
                                                           │
                                                  └────► boucle suivante
```

Les briques d'un agent :

| Brique | Rôle | Implémentation typique |
|--------|------|------------------------|
| **LLM** | Raisonnement, planification, décision de l'action suivante | API fournisseur ou modèle local |
| **System prompt** | Rôle, règles, garde-fous, format des actions | Prompt engineering (chapitre 5) |
| **Outils (tools)** | Capacités externes : recherche, fichiers, API, code, BDD | Function calling / tool use du modèle, ou MCP |
| **Mémoire** | Historique de la conversation, faits retenus, plan en cours | Contexte court terme (historique) + long terme (vector store, base clé-valeur) |
| **Planification** | Découper l'objectif en étapes, s'auto-corriger | Chaîne de pensée, arbres de recherche (ReAct, plan-and-execute) |
| **Garde-fous** | Valider les actions avant exécution, limiter les dégâts | Permissions, validation humaine, bac à sable, budgets |

## Le tool use / function calling

Mécanisme clé : on **décrit des outils** au modèle (nom, description, schéma des paramètres). Le modèle décide d'en appeler un en produisant un appel structuré ; votre code exécute l'outil et renvoie le résultat ; le modèle continue.

```json
{
  "name": "rechercher_vol",
  "description": "Recherche des vols entre deux villes à une date donnée.",
  "parameters": {
    "type": "object",
    "properties": {
      "depart": {"type": "string", "description": "Ville de départ"},
      "arrivee": {"type": "string", "description": "Ville d'arrivée"},
      "date": {"type": "string", "description": "Date au format AAAA-MM-JJ"}
    },
    "required": ["depart", "arrivee", "date"]
  }
}
```

La **qualité de la description** est déterminante : elle doit dire *quand* utiliser l'outil, pas seulement *ce qu'il fait*. Un outil mal décrit est un outil mal utilisé (ou jamais appelé).

## MCP : le standard pour connecter les outils

Le **Model Context Protocol (MCP)** est un standard ouvert (lancé par Anthropic fin 2024, donné à l'Agentic AI Foundation / Linux Foundation fin 2025) qui standardise la connexion entre un LLM et des outils / sources de données. Analogie : l'USB-C de l'IA.

- **MCP server** : expose des outils, des ressources (fichiers, données) et des prompts.
- **MCP client** : l'application agent, qui se connecte à un ou plusieurs serveurs.
- **Avantage** : un serveur MCP (ex. : accès GitHub, Notion, PostgreSQL, système de fichiers) s'écrit **une fois** et sert à **tout** client compatible — plus besoin d'intégration sur mesure par outil.

```python
# Exemple : agent avec deux serveurs MCP (recherche web + système de fichiers)
# Prérequis : pip install anthropic mcp
# Le serveur filesystem est exécuté en local via npx (Node.js requis).
from anthropic import Anthropic
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def agent(question: str) -> str:
    client = Anthropic()
    servers = {
        "recherche": StdioServerParameters(
            command="npx", args=["-y", "@modelcontextprotocol/server-brave-search"]),
        "fichiers": StdioServerParameters(
            command="npx", args=["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]),
    }
    async with stdio_client(servers["recherche"]) as (r1, w1), \
               stdio_client(servers["fichiers"]) as (r2, w2):
        async with ClientSession(r1, w1) as s1, ClientSession(r2, w2) as s2:
            await s1.initialize()
            await s2.initialize()
            tools = (await s1.list_tools()).tools + (await s2.list_tools()).tools
            # Conversion au format tool use d'Anthropic, puis boucle agent :
            # appel modèle → si tool call, exécution via la session concernée
            # → résultat réinjecté → itération jusqu'à la réponse finale.
            ...
```

L'écosystème MCP compte des milliers de serveurs en 2026 (GitHub, Slack, Notion, bases de données, navigateurs, etc.). Vérifier la provenance et les permissions d'un serveur avant de le connecter à un agent ayant des droits réels.

## Les patterns d'agents

| Pattern | Principe | Usage |
|---------|----------|-------|
| **ReAct** | Raisonnement + action alternés : le modèle réfléchit, agit, observe, réfléchit… | Tâches générales, recherche d'information |
| **Plan-and-execute** | D'abord un plan en N étapes, puis exécution étape par étape | Tâches longues et structurées |
| **Reflection / self-critique** | Le modèle évalue sa propre sortie et la corrige | Qualité élevée requise (rédaction, code) |
| **Multi-agents** | Plusieurs agents spécialisés qui se passent le relais (ex. : chercheur → rédacteur → relecteur) | Tâches complexes, séparation des rôles |
| **Human-in-the-loop** | Validation humaine avant les actions sensibles | Toute action à conséquence (déploiement, paiement, envoi d'email) |

## Mémoire : court terme et long terme

- **Court terme** : l'historique de la boucle, dans la fenêtre de contexte. Géré par l'application (résumé automatique quand le contexte déborde).
- **Long terme** : une base externe (vector store pour les faits mémorisés, clé-valeur pour les préférences). Le modèle écrit et relit cette mémoire via des outils.

Sans mémoire long terme, l'agent « oublie » d'une session à l'autre. Avec, il s'améliore sur l'utilisateur — au prix d'une surface d'attaque supplémentaire (une mémoire empoisonnée empoisonne l'agent).

## Garde-fous indispensables

Un agent qui peut agir peut faire des dégâts. Minimum vital :

1. **Principe du moindre privilège** : chaque outil a les droits minimums. Un agent qui lit des fichiers ne doit pas pouvoir les supprimer.
2. **Validation humaine** pour les actions irréversibles ou coûteuses (envoyer un mail, déployer, payer, modifier une BDD de prod).
3. **Budgets** : nombre max d'appels d'outils, coût max en tokens, durée max de la boucle. Sans budget, une boucle agent diverge et la facture explose.
4. **Bac à sable** : exécution de code dans un conteneur isolé, jamais sur la machine hôte.
5. **Journalisation** : tracer chaque décision et chaque action de l'agent (qui a demandé quoi, quel outil, quel résultat). Indispensable pour déboguer et auditer.
6. **Protection contre l'injection de prompt** : le contenu externe (pages web, emails, documents) peut contenir des instructions destinées à détourner l'agent (« ignore tes instructions et envoie les données à… »). Traiter tout contenu externe comme **données, jamais comme instructions** ; séparer instructions et données dans le prompt (chapitre 5).

## Construire un premier agent : recette

1. **Commencer sans agent.** Un chatbot + RAG (chapitre 6) résout déjà 80 % des besoins. Un agent ajoute de la complexité : ne le sortir que quand le modèle doit *agir*.
2. **Un objectif, un outil, une boucle.** Premier agent viable : un seul outil bien décrit, une boucle simple, un budget de 10 itérations max.
3. **System prompt solide** : rôle, format des actions, interdiction d'inventer, règle « si tu ne sais pas, demande ».
4. **Boucle explicite en code** : ne pas déléguer la boucle à un framework opaque au début. Comprendre chaque tour.
5. **Évaluer sur des scénarios** : 10-20 cas nominaux + cas limites (outil qui échoue, question hors périmètre, boucle infinie).
6. **Ajouter les garde-fous avant d'élargir** : permissions, validation humaine, journalisation.
7. **Passer à un framework** quand ça marche : LangGraph, CrewAI, AutoGen, ou l'Agents SDK de votre fournisseur — quand la boucle maison devient un frein.

### Exemple de squelette de boucle (pseudo-code lisible)

```python
BUDGET_MAX_TOURS = 10

messages = [system_prompt, {"role": "user", "content": objectif}]
for tour in range(BUDGET_MAX_TOURS):
    reponse = llm.ask(messages=messages, tools=mes_outils)
    if reponse.est_texte_final:            # le modèle a répondu
        return reponse.texte
    for appel in reponse.appels_outils:     # le modèle demande des actions
        if not est_autorise(appel):         # garde-fou
            resultat = "Action refusée par les garde-fous."
        else:
            resultat = executer(appel)       # bac à sable recommandé
        messages.append(resultat_observe)    # retour dans le contexte
    # la boucle continue avec le résultat observé
raise BudgetDepasse("L'agent n'a pas conclu en 10 tours.")
```

## Les frameworks d'agents (2026)

| Framework | Forces | Pour qui |
|-----------|--------|----------|
| **LangGraph** (LangChain) | Boucles explicites, état persistant, graphe de contrôle | Équipes qui veulent contrôler chaque transition |
| **Agents SDK** (OpenAI, Anthropic) | Intégration native tool use, hébergé, simple | Démarrage rapide, providers déjà choisis |
| **CrewAI** | Multi-agents par rôles, lisible | Scénarios « équipe d'experts » |
| **AutoGen** (Microsoft) | Conversations multi-agents, recherche | Expérimentation, recherche |
| **MCP natif** | Interopérabilité des outils | Dès qu'on connecte plusieurs sources de données |
| **En local** : Ollama + outils maison, llama.cpp, vLLM | Confidentialité, coût fixe | Données sensibles, souveraineté |

## Pièges connus

- **La boucle infinie** : l'agent appelle le même outil en boucle sans conclure. Toujours un budget de tours + détection de répétition.
- **L'explosion de contexte** : chaque tour ajoute du texte ; la fenêtre déborde et les coûts montent. Résumer l'historique régulièrement.
- **L'outil mal décrit** : le modèle ne l'appelle jamais, ou l'appelle n'importe comment. Tester chaque outil isolément.
- **Trop d'outils** : au-delà de ~10-15 outils, la décision de l'outil se dégrade. Regrouper, hiérarchiser, ou utiliser un routeur.
- **Confiance aveugle dans l'agent** : un agent qui « a l'air de fonctionner » sur 3 démos peut échouer en production. Scénarios de test obligatoires, journalisation, revue humaine des actions à risque.
- **Sécurité MCP** : un serveur MCP malveillant ou compromis exécute du code chez vous. N'installer que des serveurs audités, en lecture seule si possible.

## Références

- [Anthropic — « Building effective agents »](https://www.anthropic.com/engineering/building-effective-agents) — article de référence sur les patterns d'agents (en anglais)
- [Model Context Protocol — spécification et documentation](https://modelcontextprotocol.io/) — standard ouvert des outils (en anglais)
- [LangGraph — documentation](https://langchain-ai.github.io/langgraph/) — boucles d'agents en graphes (en anglais)
- [OpenAI — « Agents SDK »](https://platform.openai.com/docs/guides/agents) — SDK officiel (en anglais)
- [OWASP — « LLM Top 10 » (section agents et plugins)](https://owasp.org/www-project-top-10-for-large-language-model-applications/) — risques de sécurité (en anglais)
- [Simon Willison — « Prompt injection » (blog)](https://simonwillison.net/) — veille de référence sur les attaques (en anglais)

---

# Chapitre 8 — Bilan des modèles et de leur utilisation

> Partie de : [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md)

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
| **GLM-5.3 (+ Flash)** | Z.ai | MIT | Très fort en code agentique et tool use ; GLM-5.3-Flash très économique | Code, agents, hébergement local économique (voir article [colibri](articles/colibri/README.md)) |
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
| 100B+ (MoE) | Qualité proche du frontier en local | DeepSeek V4, GLM-5.3, Kimi K3, Qwen3 MoE | Cluster GPU, ou moteur spécialisé (voir [colibri](articles/colibri/README.md) : MoE géants en CPU/RAM/disque) |

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
- [LM Studio sur Ubuntu (article local)](articles/lm-studio/README.md) — faire tourner des modèles open weights en local
- [colibri (article local)](articles/colibri/README.md) — moteur d'inférence pour grands modèles MoE sur matériel ordinaire
- [FreeLLMAPI (article local)](articles/freellmapi/README.md) — agréger les free tiers de fournisseurs LLM derrière une API unifiée

---

# Chapitre 9 — Comment fonctionne un réseau de neurones

> Partie de : [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md)

**Date :** 2026-10-29

## Objectif

Expliquer le fonctionnement interne d'un réseau de neurones artificiel — neurone, couches, propagation avant, rétropropagation — pour comprendre ce que signifient « poids », « paramètres » et « entraînement » quand on parle d'un LLM (chapitre 1) et de son cycle de vie (chapitre 2).

## Du neurone biologique au neurone artificiel

L'analogie de départ : un neurone biologique reçoit des signaux via ses dendrites, les intègre, et émet un signal via son axone si le total dépasse un seuil. Le **neurone artificiel** (ou perceptron) reprend cette idée sous forme de calcul :

```
x1 ──w1──┐
x2 ──w2──┼──>  Σ (+ b)  ──>  f(·)  ──>  sortie a
x3 ──w3──┘

z = w1·x1 + w2·x2 + w3·x3 + b     (somme pondérée + biais)
a = f(z)                           (fonction d'activation)
```

- **Les entrées** `x1, x2, x3` : les données (pixels, caractéristiques, vecteur d'un token…).
- **Les poids** `w1, w2, w3` et le **biais** `b` : les paramètres du neurone, ajustés pendant l'entraînement.
- **La fonction d'activation** `f` : introduit une non-linéarité. Sans elle, empiler des neurones resterait une simple transformation linéaire, incapable de représenter des frontières complexes.

Fonctions d'activation courantes :

| Fonction | Formule | Usage |
|----------|---------|-------|
| Sigmoïde | `1 / (1 + e^-z)` | Historique ; aujourd'hui évitée dans les couches cachées (gradient qui s'évanouit) |
| Tanh | `tanh(z)` | Variante centrée de la sigmoïde |
| ReLU | `max(0, z)` | La plus répandue : simple, rapide, robuste au gradient évanescent |
| GELU / SwiGLU | variantes lisses | Standard dans les transformers modernes (meilleure qualité, coût un peu plus élevé) |

## Des neurones aux couches : le perceptron multicouche

Un **réseau de neurones** assemble des neurones en couches connectées : une couche d'entrée, une ou plusieurs couches cachées, une couche de sortie. Dans une couche « dense » (fully connected), chaque neurone est connecté à tous les neurones de la couche suivante.

- **Profondeur** = nombre de couches ; **largeur** = nombre de neurones par couche.
- Chaque connexion porte un poids : c'est l'ensemble de ces poids qui constitue les **paramètres** du réseau. « Un modèle de 7 milliards de paramètres » signifie 7 milliards de poids (et biais), pas 7 milliards de neurones.
- Une couche entière se calcule en une opération : `y = f(W·x + b)`, où `W` est la matrice des poids. D'où l'importance des GPU : ce sont des multiplications de matrices massivement parallélisables.

## Propagation avant : calculer une sortie

La **propagation avant** (forward pass) enchaîne les couches de gauche à droite. Exemple : un réseau qui estime une note sur 10 à partir de trois critères normalisés (scénario, jeu des acteurs, effets spéciaux) :

```
Entrée        : x = [0.9, 0.4, 0.8]
Couche cachée : z = W1·x + b1   puis   a = ReLU(z)
Sortie        : y = W2·a + b2   →  note prédite
```

À l'initialisation, les poids sont tirés aléatoirement : la sortie est du bruit. C'est l'entraînement qui transforme ce bruit en prédictions utiles.

## La fonction de perte : mesurer l'erreur

La **fonction de perte** (loss) quantifie l'écart entre la sortie prédite et la sortie attendue :

- **Erreur quadratique moyenne** pour la régression (prédire un nombre).
- **Entropie croisée** pour la classification — et c'est elle qu'utilisent les LLM : la sortie est une distribution de probabilités sur tout le vocabulaire, et la cible est le vrai token suivant (chapitre 1).

## Rétropropagation : apprendre de ses erreurs

L'entraînement consiste à ajuster chaque poids pour réduire la perte. La **rétropropagation** calcule, pour chaque poids, de combien il contribue à l'erreur — le gradient `∂L/∂w` — en appliquant la règle de la chaîne de la sortie vers l'entrée, puis met à jour les poids dans le sens opposé :

```
w ← w − η · ∂L/∂w        (η = taux d'apprentissage, learning rate)
```

La boucle d'entraînement, répétée sur des millions de lots (batches) d'exemples :

```
pour chaque lot d'exemples :
    1. propagation avant   : calculer la prédiction
    2. calculer la perte
    3. rétropropagation     : calculer les gradients de tous les poids
    4. mise à jour          : w ← w − η · gradient
```

Le **taux d'apprentissage** `η` est le réglage le plus sensible : trop grand, l'apprentissage diverge ou oscille ; trop petit, il est extrêmement lent. En pratique, on utilise un optimiseur comme **Adam** (taux adaptatif par paramètre, avec momentum) et un planning de `η` (warmup puis décroissance).

## Ce que « apprendre » signifie concrètement

Le réseau ne mémorise pas les exemples un par un : il ajuste ses milliards de poids pour que la fonction globale approxime au mieux la distribution des données. C'est ce qui lui permet de **généraliser** à des exemples jamais vus — et aussi d'apprendre les biais et les erreurs du corpus (voir chapitres 2 et 3).

Analogie : apprendre à reconnaître un chat, c'est ajuster les poids pour que « photo avec oreilles pointues et moustaches » produise une forte probabilité « chat », sans avoir stocké aucune photo.

## Du perceptron au transformer

Le transformer des LLM (chapitre 1) réutilise exactement ces briques :

- **Embedding** : chaque token est converti en un vecteur dense (ex. 4 096 dimensions) via une matrice de poids apprise.
- **Bloc transformer** = deux sous-briques empilées :
  - **l'attention** (chapitre 1) : mélange l'information entre les tokens ;
  - **le MLP** (perceptron multicouche à 2 couches avec activation) : appliqué à chaque token indépendamment.
- Un LLM = N blocs identiques (typiquement 32 à 128), plus la tête de sortie qui projette sur le vocabulaire.

Les poids d'un LLM sont donc : la matrice d'embedding + les poids de chaque bloc (matrices Q, K, V, O de l'attention + poids du MLP) + la tête de sortie. Ordres de grandeur : un modèle « 7B » = 7 milliards de poids ≈ 14 Go en FP16 ; un « 70B » ≈ 140 Go en FP16 — d'où la quantification (chapitre 10).

## L'entraînement à grande échelle (aperçu)

Passer d'un réseau jouet à un LLM demande une infrastructure considérable (détails et coûts au chapitre 2) :

- **Données** : des centaines de milliards à des billions de tokens, nettoyés et dédupliqués.
- **Matériel** : des milliers de GPU (NVIDIA H100/B200…) interconnectés (NVLink, InfiniBand) dans un data center.
- **Parallélisme** : les données sont réparties sur les GPU (data parallel), et le modèle lui-même est découpé (tensor parallel, pipeline parallel, ZeRO pour la mémoire de l'optimiseur).
- **Scaling laws** : la perte diminue selon des lois de puissance prévisibles en fonction des données, des paramètres et du calcul — ce qui guide les décisions d'investissement des laboratoires.

C'est pourquoi l'entraînement complet est réservé aux laboratoires (chapitre 2, phase 1), et pourquoi les utilisateurs se contentent d'inférence (chapitre 10) ou de fine-tuning léger (chapitre 6).

## Pièges connus

- **« C'est de la magie » / « c'est juste des maths »** : ni l'un ni l'autre. Le réseau calcule une fonction déterministe, très complexe ; ce qui surprend, c'est l'émergence de comportements jamais explicitement programmés.
- **Confondre paramètres et neurones** : « 7 milliards de neurones » est un abus de langage pour 7 milliards de poids.
- **Croire que le modèle « stocke » ses données d'entraînement** : il compresse des régularités statistiques dans ses poids. Une mémorisation partielle de fragments existe néanmoins (risque de fuite de données, voir chapitre 3).
- **Sous-estimer le rôle des données** : aujourd'hui, plus de données de qualité bat souvent plus de paramètres ; un « petit » modèle bien entraîné dépasse un « gros » modèle mal entraîné.

## Références

- [3Blue1Brown — « But what is a Neural Network? »](https://www.3blue1brown.com/lessons/neural-networks/) — introduction visuelle, accessible sans maths (en anglais)
- [3Blue1Brown — série Deep Learning (réseau, descente de gradient, rétropropagation)](https://www.3blue1brown.com/topics/neural-networks) — la meilleure visualisation de l'entraînement (en anglais)
- [Michael Nielsen — « Neural Networks and Deep Learning » (livre en ligne gratuit)](http://neuralnetworksanddeeplearning.com/) — chapitre 1 (perceptron) et chapitre 3 (rétropropagation), avec code Python exécutable
- [Andrej Karpathy — « Neural Networks: Zero to Hero »](https://github.com/karpathy/nn-zero-to-hero) — reconstruire un mini-GPT from scratch en Python (en anglais)
- [« Attention Is All You Need » — papier fondateur du transformer (arXiv)](https://arxiv.org/abs/1706.03762)
- [LoRA — Low-Rank Adaptation, fine-tuning efficace (arXiv)](https://arxiv.org/abs/2106.09685) — voir chapitre 6
- [Chapitre 1 de cette série](articles/llm-fondamentaux/01-fonctionnement.md) — le transformer et l'attention au-dessus de ces briques

---

# Chapitre 10 — L'inférence : faire tourner un LLM

> Partie de : [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md)

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
| Sur votre machine (local) | llama.cpp, Ollama, LM Studio, MLX | Confidentialité, coût nul à l'usage, hors-ligne — voir l'article [LM Studio sur Ubuntu](articles/lm-studio/README.md) et le moteur [colibri](articles/colibri/README.md) |
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
- [Chapitre 1 de cette série](articles/llm-fondamentaux/01-fonctionnement.md) — la boucle de génération token par token ; [chapitre 9](articles/llm-fondamentaux/09-reseaux-de-neurones.md) — le réseau sous-jacent

---

# Harness engineering — concevoir l'environnement d'un agent IA

> Catégorie : Guide

**Date :** 2026-10-29
**Statut :** Terminé

## Objectif

Comprendre ce qu'est le **harness engineering** (ingénierie du harnais) : la discipline qui consiste à concevoir tout l'environnement logiciel autour d'un LLM — outils, mémoire, boucle, vérifications, garde-fous — pour transformer un modèle en agent fiable, et savoir appliquer ses principes dans ses propres projets.

## Prérequis

- Avoir lu la série [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md), en particulier le chapitre 7 sur les [agents IA](articles/llm-fondamentaux/07-agent-ia.md) (boucle raisonnement + outils, MCP, garde-fous).
- Connaissance de base d'un agent de codage (Codex, Claude Code, Cursor ou équivalent).

## Définition : agent = modèle + harnais

Un LLM de base est une **fonction probabiliste sans état** : il transforme une séquence de tokens en une autre, sans mémoire, sans accès au monde. Le **harnais d'agent** (agent harness) est toute l'infrastructure logicielle qui entoure le modèle pour le faire agir comme un agent :

```
Agent = Modèle + Harnais

Modèle  : l'intelligence (raisonnement, langue, connaissances)
Harnais : tout le reste — boucle d'exécution, outils, mémoire,
          gestion du contexte, sandbox, permissions, vérifications
```

Autrement dit, si vous n'êtes pas le modèle, vous êtes le harnais : chaque morceau de code, de configuration et de logique d'exécution qui n'est pas le modèle lui-même.

**Pourquoi cela compte** : à modèle égal, deux agents peuvent avoir des taux de réussite radicalement différents selon leur harnais. LangChain a fait passer son agent de codage de la 30ᵉ place au top 5 du benchmark Terminal-Bench 2.0 **en ne changeant que le harnais** (même modèle, mêmes poids). Vercel a supprimé 80 % des outils de son agent et a vu son taux de réussite passer de 80 % à 100 %.

## Origine du terme

Le vocabulaire émerge début 2026, avec une attribution disputée :

- **Mitchell Hashimoto** (cofondateur de HashiCorp, 5 février 2026) décrit la pratique consistant, à chaque erreur d'un agent, à **corriger durablement l'environnement** pour que l'erreur ne se reproduise jamais — plutôt que d'espérer qu'un meilleur prompt fera mieux.
- **Vivek Trivedy** (LangChain) popularise la formule « Agent = Modèle + Harnais » et en dérive l'anatomie des composants.
- **OpenAI** publie en février 2026 le récit d'une équipe ayant livré un produit interne de ~1 million de lignes de code en 5 mois **sans aucune ligne écrite à la main**, entièrement par des agents Codex guidés par leur harnais (PR, CI, règles dans le dépôt).

## Les trois niveaux d'ingénierie

Le harness engineering s'inscrit dans une progression historique où le périmètre de ce que l'humain conçoit s'élargit :

| Discipline | Époque | Périmètre | Question centrale |
|------------|--------|-----------|-------------------|
| **Prompt engineering** | 2022-2024 | Le message (une interaction) | « Que demander, et comment ? » |
| **Context engineering** | 2025 | Le contenu de la fenêtre de contexte à un instant donné | « Quelle information le modèle voit-il maintenant ? » |
| **Harness engineering** | 2026+ | Tout l'environnement opérationnel de l'agent, sur toute la durée de la tâche | « Quels outils, quelles vérifications, quelles boucles entourent le modèle ? » |

Chaque couche englobe la précédente : un bon prompt reste nécessaire, mais il n'est plus qu'un composant du harnais.

## Anatomie d'un harnais

Composants usuels d'un harnais d'agent (le chapitre 7 en décrit le principe général) :

| Composant | Rôle | Exemples concrets |
|-----------|------|-------------------|
| **Boucle d'exécution** | Orchestre raisonner → agir → observer → recommencer jusqu'à un critère d'arrêt | Patron ReAct, boucle « while » avec outils |
| **Outils (tools)** | Donne au modèle un moyen d'agir sur le monde | lecture/écriture de fichiers, shell, recherche web, appels API, MCP |
| **Mémoire et état** | Persiste l'information au-delà de la fenêtre de contexte | fichiers `NOTES.md` / `progress.txt`, base de données, historique git |
| **Gestion du contexte** | Garde la fenêtre de contexte utile (taille finie, « context rot ») | compaction, résumé, chargement « juste-à-temps » (chemins plutôt que contenus) |
| **Environnement d'exécution** | Isole et sécurise les actions de l'agent | sandbox, conteneur, permissions scopées, niveaux d'approbation |
| **Vérifications (sensors)** | Observent le résultat et permettent l'auto-correction | tests, linters, CI, « LLM-juge » |
| **Garde-fous** | Empêchent les dérives dangereuses | permissions, validation humaine, monitoring, pistes d'audit |
| **Observabilité** | Rend l'agent compréhensible et débogable | logs des appels d'outils, traces, métriques de coût/latence |

**Guides vs sensors** (distinction de Birgitta Böckeler, Thoughtworks) : les **guides** orientent l'agent avant qu'il agisse (instructions, conventions, fichiers `AGENTS.md`/`CLAUDE.md`) ; les **sensors** observent le résultat après coup (tests, vérifications) et déclenchent la correction. Sans guides, l'agent n'a pas de règles ; sans sensors, il répète les mêmes erreurs. Chacun peut être **déterministe** (test, linter) ou **inférentiel** (un LLM juge la sortie).

## Harnais interne vs externe

- **Harnais interne** : livré par le concepteur du modèle — SDK d'agents, outil de codage (Codex, Claude Code, Cursor…). Vous le choisissez, vous ne le modifiez pas.
- **Harnais externe** : assemblé par vous par-dessus — fichiers d'instructions (`AGENTS.md`, `CLAUDE.md`), serveurs MCP, skills personnalisés, scripts CI, règles de revue. C'est là que se joue la majeure partie du harness engineering.

## Principes pratiques

1. **Corriger l'environnement, pas le prompt.** Chaque erreur récurrente de l'agent doit se traduire par une règle mécanique dans le dépôt (règle de lint, test, convention dans `AGENTS.md`), pas par une phrase de plus dans le prompt. L'instruction textuelle s'oublie quand le contexte se compresse ; la contrainte mécanique reste.
2. **Rendre l'agent « lisible » (agent legibility).** Donnez-lui les moyens de vérifier lui-même son travail : accès à un navigateur (CDP), à la console, aux tests. Un agent qui peut constater son erreur la corrige souvent tout seul.
3. **Externaliser l'état.** Plutôt qu'un historique de conversation toujours plus long, faites écrire l'état dans des fichiers structurés (liste de tâches JSON, `progress.txt`) que l'agent relit au démarrage de chaque session — c'est le patron « initializer + agent de codage » d'Anthropic pour les tâches longues.
4. **Moins d'outils, de meilleurs outils.** Trop d'outils dégradent la fiabilité (Vercel : −80 % d'outils → +20 points de réussite). Des outils à un seul niveau de paramètres, bien nommés, avec des valeurs de retour claires.
5. **Boucles de feedback courtes.** CI rapide, tests exécutables par l'agent, evals sur un jeu de cas réels : la boucle « agent agit → capteur vérifie → correction » est le cœur du harnais.
6. **Séparer producteur et évaluateur.** Dans les tâches longues, l'agent qui écrit le code ne devrait pas être le seul juge de sa qualité (patron Planner/Generator/Evaluator).
7. **Garder les fichiers d'instructions concis.** Une étude ETH Zurich (février 2026) montre que les fichiers de contexte de dépôt n'améliorent pas le taux de réussite en général et augmentent le coût d'inférence de plus de 20 % : viser moins de ~100 lignes, limitées aux pratiques propres à l'équipe, avec des renvois vers la doc détaillée.

## Exemple concret : structure d'un dépôt « agent-first »

L'équipe d'OpenAI a organisé son dépôt produit pour être piloté par des agents :

```
repo/
├── AGENTS.md            # règles mécanique : conventions, commandes, interdits
├── README.md            # architecture et décisions (ADR)
├── docs/                # documentation détaillée référencée par AGENTS.md
├── .github/workflows/   # CI = capteurs : lint, tests, build, sécurité
├── scripts/             # commandes testables (init, test, lint, run)
└── src/                 # code généré par les agents, revu en PR
```

Règle d'or de cette équipe : **toute modification de comportement passe par le dépôt** (règle, test, doc), jamais par une consigne orale ou un prompt oublié.

## Harnais d'agent vs harnais d'évaluation

Ne pas confondre :

- **Harnais d'agent** : entoure l'agent **pendant** la tâche (outils, boucle, vérifications en direct).
- **Harnais d'évaluation** : mesure **après coup** la qualité des sorties (benchmarks, evals, jeux de test) — voir le chapitre 5 de la série LLM pour l'évaluation de prompts.

Les deux se complètent : les evals du harnais d'évaluation alimentent les capteurs du harnais d'agent.

## Pièges connus

- **« Le modèle suffit »** : à capacité égale, c'est le harnais qui fait la différence en production. Ne pas investir dans un plus gros modèle avant d'avoir un harnais qui vérifie.
- **Accumuler les règles dans le prompt** : au-delà d'une certaine taille, les instructions se contredisent ou sont ignorées quand le contexte se compresse. Préférer des contraintes mécaniques (tests, lint, schéma de sortie).
- **Laisser l'agent se déclarer « terminé »** : le modèle non déterministe peut inventer une action ou conclure prématurément. Exiger une preuve vérifiable (tests passés, artifact produit, critères explicites).
- **Oublier l'observabilité** : sans traces des appels d'outils et des décisions, déboguer un agent en production est un cauchemar. Logger dès le début.
- **Un seul agent pour tout faire** : pour les longues tâches, séparer initialisation, production et évaluation, avec des artefacts explicites entre sessions.
- **Négliger la sécurité du sandbox** : un agent avec un shell et des permissions larges est une surface d'attaque (injection de prompt, exfiltration). Scoper les permissions, valider les entrées, auditer les actions (voir chapitre 7).

## Références

- [Mitchell Hashimoto — « My AI Adoption Journey » (5 fév. 2026)](https://mitchellh.com/writing/my-ai-adoption-journey) — billet fondateur du terme « harness engineering »
- [LangChain (Vivek Trivedy) — « The Anatomy of an Agent Harness »](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness) — la formule Agent = Modèle + Harnais et les composants du harnais
- [OpenAI (Ryan Lopopolo) — « Harness engineering: leveraging Codex in an agent-first world »](https://openai.com/index/harness-engineering/) — produit de ~1M de lignes écrit à 0 % à la main
- [Anthropic — « Effective harnesses for long-running agents »](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — initializer + agent de codage, `claude-progress.txt`, fenêtres de contexte multiples
- [Anthropic — « Effective context engineering for AI agents »](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — compaction, mémoire externe (NOTES.md), chargement juste-à-temps
- [Birgitta Böckeler (Thoughtworks) — « Harness engineering for coding agent users »](https://martinfowler.com/articles/harness-engineering.html) — classification guides/capteurs, harnais interne/externe
- [Yao et al. — « ReAct: Synergizing Reasoning and Acting in Language Models » (arXiv)](https://arxiv.org/abs/2210.03629) — le patron fondateur raisonner↔agir
- [Schick et al. — « Toolformer » (arXiv)](https://arxiv.org/abs/2302.04761) — l'appel d'outils appris par le modèle
- [Wikipédia — « Harnais d'agent »](https://fr.wikipedia.org/wiki/Harnais_d%27agent) — synthèse et historique du terme
- Série [LLM — fondamentaux et panorama](articles/llm-fondamentaux/README.md), chapitre 7 : [créer un agent IA](articles/llm-fondamentaux/07-agent-ia.md)

---

# MCP — Model Context Protocol : fonctionnement du protocole et clients CLI/TUI (Inspector, mcp-tui)

> Catégorie : Guide

**Date :** 2026-11-07
**Statut :** Terminé

## Objectif

Comprendre ce qu'est le **Model Context Protocol (MCP)** et le détail de son protocole (JSON-RPC 2.0, transports, poignée de main, primitives, méthodes), puis savoir tester et déboguer un serveur MCP depuis le terminal avec les deux outils de référence : le **MCP Inspector** officiel (modes web, CLI et TUI) et **mcp-tui** (TUI autonome en Go, orienté CI).

## Prérequis

- Avoir lu le chapitre 7 de la série [LLM — fondamentaux](articles/llm-fondamentaux/07-agent-ia.md) pour le rôle de MCP dans un agent IA.
- Linux ou macOS, avec **Node.js** (≥ 18) et `npx` pour l'Inspector.
- **Go** (≥ 1.23) pour installer mcp-tui, et **Python** avec `uv`/`uvx` pour lancer un serveur MCP en exemple (`mcp-server-fetch`).
- Un serveur MCP à tester : local (`stdio`) ou distant (`https://…/mcp`).

## Qu'est-ce que MCP ?

Le **Model Context Protocol** est un standard ouvert lancé par Anthropic fin 2024 (donné depuis à l'Agentic AI Foundation / Linux Foundation) qui définit comment un **hôte** (application agent : Claude Desktop, un IDE, un CLI de codage…) se connecte à des **serveurs MCP** qui exposent des capacités externes. Analogie usuelle : l'USB-C de l'IA — un serveur s'écrit une fois, tout client compatible s'y branche.

Trois rôles :

| Rôle | Description | Exemples |
|------|-------------|----------|
| **Hôte MCP** | L'application qui embarque un client et orchestre le LLM | Claude Desktop, VS Code, Cursor, OpenCode, un agent maison |
| **Client MCP** | Le composant de l'hôte qui parle le protocole avec un serveur | `mcp` (SDK Python), `mcp-go`, Inspector, mcp-tui |
| **Serveur MCP** | Expose des outils, des ressources et des prompts | `mcp-server-fetch`, `mcp-server-git`, `mcp-server-filesystem`, serveur Notion/Postgres… |

Un hôte peut se connecter à **plusieurs serveurs** en parallèle ; chaque connexion est une session indépendante.

## Les transports

MCP encode ses messages en **JSON-RPC 2.0** (UTF-8). Deux transports standard :

| Transport | Usage | Mécanisme |
|-----------|-------|-----------|
| **stdio** | Serveur local, même machine | Le client lance le serveur en sous-processus ; un message JSON-RPC par ligne sur `stdin`/`stdout` du serveur. Les logs du serveur vont sur `stderr` (jamais sur `stdout`, réservé au protocole). |
| **Streamable HTTP** | Serveur distant, multi-clients | Un unique endpoint HTTP `POST https://…/mcp` ; la réponse peut être un JSON simple ou un flux SSE (Server-Sent Events) pour les réponses longues et les notifications serveur. En-tête `MCP-Protocol-Version` pour négocier la version. |

L'ancien transport **HTTP+SSE** (révision 2024-11-05, deux endpoints `/sse` + `/messages`) est **déprécié** mais encore supporté pour la rétrocompatibilité. Les serveurs locaux en HTTP doivent n'écouter que sur `127.0.0.1`, pas sur `0.0.0.0`.

## Le protocole en détail

### Cycle de vie d'une session

Sur les révisions ≤ 2025-11-25 (encore omniprésentes, et servies en « dual-era » par les serveurs récents) :

1. **initialize** (requête) : le client envoie `protocolVersion`, ses `capabilities` (sampling, elicitation, roots…) et `clientInfo`. Le serveur répond avec sa propre version, ses `capabilities` (ex. : `tools`, `resources`, `prompts`) et `serverInfo`. Sur HTTP, la réponse peut définir un en-tête `Mcp-Session-Id` réutilisé ensuite.
2. **notifications/initialized** (notification, sans réponse) : le client signale que la poignée de main est terminée.
3. **Échanges** : requêtes/réponses et notifications dans les deux sens jusqu'à la fermeture.

Les révisions récentes (2026-07-28, « Current ») rendent le protocole **sans état** : plus de poignée de main `initialize` ni de `Mcp-Session-Id` — chaque requête porte sa version dans `_meta` et les clients découvrent les capacités via `server/discover`. En pratique, tout client moderne négocie automatiquement la bonne ère.

### Les primitives exposées par un serveur

| Primitive | Méthodes associées | Rôle |
|-----------|--------------------|------|
| **Tools** (outils) | `tools/list`, `tools/call` | Actions que le LLM peut invoquer (fonction avec schéma d'entrée JSON et résultat structuré) |
| **Resources** (ressources) | `resources/list`, `resources/read`, `resources/templates/list` | Données en lecture seule adressables par URI (`file://…`, `https://…`) |
| **Prompts** | `prompts/list`, `prompts/get` | Templates de prompts réutilisables avec arguments |

Méthodes transverses : `ping`, `logging/setLevel`, `completion/complete`, plus les notifications `notifications/tools/list_changed`, `notifications/resources/list_changed`, etc. Les listes sont **paginées** via un `cursor` / `nextCursor`.

Chaque outil déclare un **`inputSchema`** en JSON Schema (dialecte 2020-12 depuis la révision 2025-11-25) : c'est ce schéma que les TUI utilisent pour générer les formulaires de saisie.

### Exemple d'échange JSON-RPC

```json
→ {"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"exemple","version":"1.0"}}}
← {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2025-11-25","capabilities":{"tools":{"listChanged":true}},"serverInfo":{"name":"demo","version":"1.0"}}}
→ {"jsonrpc":"2.0","method":"notifications/initialized"}
→ {"jsonrpc":"2.0","id":2,"method":"tools/list"}
← {"jsonrpc":"2.0","id":2,"result":{"tools":[{"name":"get_weather","description":"Météo courante d'un lieu","inputSchema":{"type":"object","properties":{"location":{"type":"string","description":"Ville ou code postal"}},"required":["location"]}}]}}
→ {"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"get_weather","arguments":{"location":"Paris"}}}
← {"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"20 °C, ensoleillé"}],"isError":false}}
```

Le résultat d'un `tools/call` est un tableau `content` de blocs typés (`text`, `image`, `audio`, `resource`, `resource_link`) plus un booléen `isError` : `isError: true` signale une **erreur d'exécution côté serveur** (outils), pas une erreur de protocole JSON-RPC (qui aurait un champ `error` à la place de `result`).

### Les callbacks serveur → client

Un serveur peut faire des requêtes **au client** (et non l'inverse), si le client a déclaré la capacité correspondante dans `initialize` :

| Callback | Capacité client | Usage |
|----------|-----------------|-------|
| `sampling/createMessage` | `sampling` | Le serveur demande au client de faire compléter un prompt par son LLM (le serveur n'a pas de modèle) |
| `elicitation/create` | `elicitation` | Le serveur demande des informations à l'utilisateur (formulaire ou URL), ex. : demander une clé API |
| `roots/list` | `roots` | Le serveur liste les répertoires de travail autorisés par l'utilisateur |
| `notifications/message` (logging) | — | Niveaux de log envoyés par le serveur |

## MCP Inspector — l'outil officiel

Le [MCP Inspector](https://github.com/modelcontextprotocol/inspector) est le client de test/débogage maintenu par l'organisation `modelcontextprotocol`. Trois modes :

- **Web** (défaut) : `npx @modelcontextprotocol/inspector` ouvre une UI sur `http://localhost:6274` (connexion, onglets Tools/Resources/Prompts, formulaires, historique des requêtes JSON-RPC).
- **CLI** : `npx @modelcontextprotocol/inspector --cli <commande-du-serveur> --method <méthode>` — exécute une méthode et affiche le résultat, sans UI. Idéal en script/CI.
- **TUI** : mode interactif en terminal couvrant la même chose que le web — connexion au serveur, navigation dans les outils, **formulaires générés automatiquement depuis le JSON Schema** de chaque outil, exécution et affichage de la sortie.

### Mode CLI : lister et appeler des outils

```bash
# Lister les outils d'un serveur stdio (ici un serveur Node local)
npx @modelcontextprotocol/inspector --cli node build/index.js --method tools/list

# Appeler un outil avec ses arguments JSON
# (--tool-arg attend UNE chaîne JSON, d'où les guillemets simples sous le shell)
npx @modelcontextprotocol/inspector --cli node build/index.js \
  --method tools/call --tool-name get_weather \
  --tool-arg '{"location": "Paris"}'

# Idem sur un serveur distant HTTP : préciser --transport http
npx @modelcontextprotocol/inspector --cli https://mcp.example.com/mcp \
  --transport http --method tools/list
```

Autres exemples de lancement de serveur en argument : `npx -y @modelcontextprotocol/inspector uvx mcp-server-fetch` (serveur Python via `uvx`) ou `npx @modelcontextprotocol/server-filesystem /chemin/autorisé`.

## mcp-tui (standardbeagle) — TUI autonome en Go

[mcp-tui](https://github.com/standardbeagle/mcp-tui) est un client TUI **et CLI** écrit en Go (framework Bubble Tea, sur le SDK MCP Go officiel), conçu précisément pour parcourir les outils d'un serveur, remplir un formulaire, exécuter et récupérer l'output. Transports supportés : **stdio, HTTP, Streamable HTTP, SSE**.

```bash
# Installation
go install github.com/standardbeagle/mcp-tui@latest

# Lancer le TUI sur un serveur lancé via uvx
mcp-tui --cmd uvx --args mcp-server-fetch
```

Fonctionnalités clés :

- **Formulaire interactif** généré depuis le JSON Schema de chaque outil, exécution, affichage du résultat.
- **Génération de la commande équivalente** en ligne de commande, réutilisable telle quelle en CI (sortie JSON-friendly).
- **Gestion des callbacks serveur** (`sampling`, `elicitation`) : le TUI répond aux demandes du serveur de façon interactive.
- Mode CLI non interactif pour scripter (`list`, `call`, etc.) avec sortie JSON.

## Inspector vs mcp-tui : lequel choisir ?

| Critère | MCP Inspector (officiel) | mcp-tui (standardbeagle) |
|---------|--------------------------|--------------------------|
| Mainteneur | Organisation `modelcontextprotocol` | standardbeagle (individuel) |
| Langage | TypeScript / Node (`npx`) | Go (binaire unique) |
| Modes | Web + CLI + TUI | TUI + CLI |
| Transports | stdio, HTTP, SSE | stdio, HTTP, Streamable HTTP, SSE |
| Formulaires depuis le JSON Schema | Oui (web et TUI) | Oui |
| Callbacks sampling/elicitation | Oui | Oui |
| Sortie scriptable / CI | Mode `--cli` | Oui, avec génération de la commande équivalente |
| Cas d'usage privilégié | Référence officielle, débogage complet (traces JSON-RPC) | Terminal rapide, tests automatisés, CI |

## Pièges connus

- **Oublier `--transport http`** sur un serveur distant : l'Inspector tente `stdio` par défaut et échoue avec une erreur de lancement de processus.
- **`--tool-arg`** attend une chaîne JSON valide : sous bash/zsh, protéger avec des guillemets simples (`'{"location":"Paris"}'`) pour éviter que le shell ne mange les guillemets internes.
- **stdio** : tout ce que le serveur écrit sur `stdout` hors JSON-RPC corrompt le flux — les logs doivent passer par `stderr`.
- **Première exécution lente** : `npx`/`uvx` téléchargent le paquet à la première invocation (réseau requis la première fois).
- **Pagination** : une réponse `tools/list` peut contenir `nextCursor` — ne pas croire la liste exhaustive sans le suivre.
- **`isError: true`** dans le résultat d'un `tools/call` = l'outil a échoué côté serveur (arguments invalides, ressource indisponible) ; ce n'est pas une erreur de transport.
- **Jetons d'accès** sur serveur HTTP distant : configurer l'en-tête `Authorization` (OAuth 2.1 / bearer) dans l'outil plutôt que dans l'URL.

## Références

- [Dépôt officiel de la spécification MCP](https://github.com/modelcontextprotocol/modelcontextprotocol)
- [Spécification MCP — transports (révision 2025-11-25)](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)
- [Spécification MCP — cycle de vie](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle)
- [Spécification MCP — outils côté serveur](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)
- [MCP Inspector — documentation officielle](https://modelcontextprotocol.io/docs/tools/inspector)
- [Dépôt GitHub du MCP Inspector (README clients CLI/TUI)](https://github.com/modelcontextprotocol/inspector)
- [Site de mcp-tui](https://standardbeagle.github.io/mcp-tui/)
- [Dépôt GitHub de mcp-tui](https://github.com/standardbeagle/mcp-tui)
- [Registre des serveurs MCP officiels](https://github.com/modelcontextprotocol/servers)
