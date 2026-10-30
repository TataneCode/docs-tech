# Harness engineering — concevoir l'environnement d'un agent IA

> Catégorie : Guide

**Date :** 2026-10-29
**Statut :** Terminé

## Objectif

Comprendre ce qu'est le **harness engineering** (ingénierie du harnais) : la discipline qui consiste à concevoir tout l'environnement logiciel autour d'un LLM — outils, mémoire, boucle, vérifications, garde-fous — pour transformer un modèle en agent fiable, et savoir appliquer ses principes dans ses propres projets.

## Prérequis

- Avoir lu la série [LLM — fondamentaux et panorama](../llm-fondamentaux/README.md), en particulier le chapitre 7 sur les [agents IA](../llm-fondamentaux/07-agent-ia.md) (boucle raisonnement + outils, MCP, garde-fous).
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
- Série [LLM — fondamentaux et panorama](../llm-fondamentaux/README.md), chapitre 7 : [créer un agent IA](../llm-fondamentaux/07-agent-ia.md)
