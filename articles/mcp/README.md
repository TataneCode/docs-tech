# MCP — Model Context Protocol : fonctionnement du protocole et clients CLI/TUI (Inspector, mcp-tui)

> Catégorie : Guide

**Date :** 2026-11-07
**Statut :** Terminé

## Objectif

Comprendre ce qu'est le **Model Context Protocol (MCP)** et le détail de son protocole (JSON-RPC 2.0, transports, poignée de main, primitives, méthodes), puis savoir tester et déboguer un serveur MCP depuis le terminal avec les deux outils de référence : le **MCP Inspector** officiel (modes web, CLI et TUI) et **mcp-tui** (TUI autonome en Go, orienté CI).

## Prérequis

- Avoir lu le chapitre 7 de la série [LLM — fondamentaux](../llm-fondamentaux/07-agent-ia.md) pour le rôle de MCP dans un agent IA.
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
