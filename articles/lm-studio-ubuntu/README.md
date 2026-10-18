# Héberger des LLM légers avec LM Studio sur un ThinkPad T14 sous Ubuntu

> Catégorie : Guide d'installation

**Date :** 2026-10-18
**Statut :** Brouillon

## Objectif

Installer LM Studio sur Ubuntu (base Debian) et héberger des modèles légers — en particulier Qwen — sur un ThinkPad T14 (Ryzen 5 PRO 5450U, 32 Go de RAM, sans GPU dédié), en mode natif et/ou Docker, avec un serveur API local réutilisable par d'autres projets.

## Prérequis

- Ubuntu (base Debian), compte utilisateur avec `sudo` pour l'installation native
- ~10 Go d'espace disque libre (application + modèles)
- Machine cible de référence : ThinkPad T14 Gen 2 AMD, Ryzen 5 PRO 5450U (6c/12t, Zen 3), iGPU Vega 7, 32 Go DDR4-3200
- Aucun GPU dédié : l'inférence se fait sur CPU, le modèle vit en RAM système

## Ce qu'est LM Studio

LM Studio est une application de bureau basée sur llama.cpp qui permet de chercher, télécharger et exécuter des LLM localement au format GGUF (depuis Hugging Face), sans dépendre d'un service cloud. Elle fournit :

- une interface graphique de chat avec réglages de sampling,
- la recherche et le téléchargement de modèles directement dans l'app,
- un serveur API local compatible OpenAI (`http://localhost:1234/v1`).

Le daemon headless `llmster` (LM Studio sans GUI) permet aussi un usage serveur, en natif ou en conteneur — voir la section Déploiement Docker.

## Lire sa configuration matérielle

Avant tout, vérifier la machine :

```bash
lscpu | grep -E "Model name|Thread|Core"
free -h
```

Pourquoi la config change tout :

- **32 Go de RAM sans GPU dédié** : le modèle est chargé en RAM système, partagée avec l'OS et vos applications. Le fichier GGUF ne doit pas dépasser ~60 % de la RAM libre pour rester confortable.
- **iGPU Vega 7** : il partage la même RAM. L'accélération iGPU n'apporte quasiment rien face au CPU sur cette génération ; ne pas compter dessus.
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

## Choisir ses modèles : Qwen sur 32 Go en CPU

Règle empirique : **taille du fichier GGUF ≈ RAM nécessaire × 1,2** (marge pour le contexte et l'OS).

| Modèle | Quantif. | Taille GGUF | Verdict sur T14 |
|--------|----------|-------------|-----------------|
| Qwen2.5-1.5B-Instruct | Q4_K_M | ~1 Go | Très réactif, idéal usage quotidien léger |
| Qwen2.5-3B-Instruct | Q4_K_M | ~2 Go | Réactif, bon compromis qualité/vitesse ✓ |
| Qwen2.5-7B-Instruct | Q4_K_M | ~4,7 Go | Le sweet spot de cette machine ✓ |
| Qwen2.5-14B-Instruct | Q4_K_M | ~9 Go | Possible à 32 Go, nettement plus lent — à tester |
| Qwen2.5-32B et plus | — | 20 Go+ | À éviter : swap, expérience dégradée |

Conseils :

- Privilégier les quantifications **Q4_K_M** : bon compromis qualité/taille. Éviter Q2/Q3 (cohérence dégradée) sauf contrainte forte.
- Un seul modèle chargé à la fois en mémoire.
- Sur CPU, viser 10–20 tokens/seconde pour un 7B ; si c'est plus lent, vérifier la RAM libre et le double canal.

## Premier modèle et chat

1. Onglet de recherche (loupe), taper `Qwen2.5 7B`.
2. Choisir un dépôt GGUF (ex. dépôt officiel Qwen ou unsloth), puis la quantification `Q4_K_M`.
3. Télécharger (~5 Go), puis charger le modèle dans l'onglet chat.
4. Ajuster la longueur de contexte : rester modeste (4096–8192 tokens) pour ne pas manger la RAM inutilement.
5. Tester un prompt, surveiller la vitesse de génération affichée.

## Serveur API local

Pour exposer le modèle à d'autres applications (le cœur de l'usage transverse) :

1. Onglet **Developer** → activer **Start Server**. L'endpoint est `http://localhost:1234/v1`, protocole OpenAI.
2. Définir un jeton API dans les paramètres du serveur si autre chose que le localhost doit y accéder.

Test :

```bash
curl http://localhost:1234/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5-7b-instruct",
    "messages": [{"role": "user", "content": "Bonjour !"}]
  }'
```

N'importe quel SDK OpenAI (Python, LangChain, Continue.dev...) fonctionne en pointant `base_url` vers `http://localhost:1234/v1`.

## Déploiement Docker

Pour intégrer LM Studio à un parc de conteneurs (gestion transverse des projets), deux voies :

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

Les modèles peuvent être déposés directement dans le volume (`./lmstudio-models`) au format GGUF, ou gérés via la CLI `lms`.

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
- Hybridre pragmatique : AppImage native pour la découverte et le test de modèles en GUI + Option A pour le service API permanent. Les deux peuvent partager le même dossier de modèles monté en volume.

Limites à connaître :

- L'image `llmster-preview` est en preview : épingler une version précise plutôt que `latest` en usage durable.
- Pas d'accélération GPU en conteneur sur cette machine (pas de GPU dédié) : les perfs conteneur ≈ perfs natives.
- Attention à la RAM : le conteneur + l'OS + vos apps partagent les 32 Go.

## Pièges connus

- **AppImage ne lance pas** : `libfuse2` manquant (voir installation).
- **Génération très lente** : modèle trop gros → swap. Vérifier `free -h` pendant l'inférence, descendre de taille.
- **Réponses incohérentes** : quantification trop agressive (Q2/Q3). Remonter à Q4_K_M.
- **Contexte par défaut court** : LM Studio charge souvent avec 4096 tokens par défaut ; ajuster selon le besoin en RAM.
- **Deux modèles en même temps** : le second évince le premier — planifier le chargement dans les scripts.
- **Conteneur qui tue l'ORM** : un conteneur sans limite mémoire (`mem_limit`) peut mettre la machine à genoux ; fixer des limites dans le compose.

## Références

- [Site officiel LM Studio](https://lmstudio.ai)
- [LM Studio headless (llmster) — documentation officielle](https://lmstudio.ai/docs/developer/core/headless)
- [Image Docker officielle llmster-preview](https://hub.docker.com/r/lmstudio/llmster-preview)
- [Image LinuxServer.io lm-studio](https://docs.linuxserver.io/images/docker-lm-studio/)
- [Modèles Qwen GGUF sur Hugging Face](https://huggingface.co/Qwen)
