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
- Article voisin : [colibri — Qwen3-Coder-30B-A3B en natif sur 32 Go](../colibri/README.md) (même modèle, moteur C pur sans Ollama)
