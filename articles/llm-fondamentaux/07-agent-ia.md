# Chapitre 7 — Créer un agent IA

> Partie de : [LLM — fondamentaux et panorama](README.md)

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
