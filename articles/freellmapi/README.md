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
