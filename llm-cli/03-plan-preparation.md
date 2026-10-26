# Plan de préparation — skills pour Gemini CLI et Vibe CLI / Vibe Code

Objectif : installer les 14 skills, 7 personas et 6 commandes de ce dossier sur les deux CLI, avec un résultat vérifiable à chaque étape.

## Prérequis

- Gemini CLI installé (`gemini --version`), authentifié.
- Vibe CLI installé (`vibe --version`), authentifié (compte Mistral ou clé API).
- Avoir lu [01-synthese-sources.md](01-synthese-sources.md) et [02-formats-cibles.md](02-formats-cibles.md).

## Étape 1 — Installer les 18 skills (identique pour les deux CLI)

Les skills suivent le standard Agent Skills : un seul jeu de fichiers sert aux deux CLI.

```bash
# Depuis la racine de ce dépôt
# Gemini CLI (scope utilisateur)
mkdir -p ~/.gemini/skills
cp -r llm-cli/skills/* ~/.gemini/skills/

# Vibe CLI / Vibe Code (scope utilisateur)
mkdir -p ~/.vibe/skills
cp -r llm-cli/skills/* ~/.vibe/skills/
```

**Vérification Gemini CLI** : lancer `gemini`, puis demander « liste les skills disponibles » — les 18 noms doivent apparaître. Tester l'activation : « écris un plan pour ajouter une validation d'email » doit déclencher `writing-plans`.

**Vérification Vibe CLI** : lancer `vibe`, taper `/` — les 18 skills doivent apparaître dans l'autocomplétion (elles sont `user-invocable` par défaut). Tester `/karpathy-guidelines explique-toi sur ce dépôt`.

## Étape 2 — Installer les personas (Vibe uniquement)

Gemini CLI n'a pas de mécanisme de personas ; sa couverture passe par les commandes de l'étape 4.

```bash
mkdir -p ~/.vibe/agents ~/.vibe/prompts
cp llm-cli/agents/*.toml ~/.vibe/agents/
cp llm-cli/prompts/*.md ~/.vibe/prompts/
```

**Vérification Vibe CLI** : `vibe --agent code-reviewer` (ou `/agent code-reviewer` en session) doit démarrer un sous-agent avec le prompt de revue. Tester : « lance l'agent debugger sur ce test qui échoue ».

## Étape 3 — Contexte permanent : règles Karpathy + style de sortie (les deux CLI)

Les règles comportementales de `karpathy-guidelines` et le style `i-have-adhd` sont utiles **à chaque session**, pas seulement quand le skill est invoqué. On les installe comme contexte permanent :

- **Vibe** : `~/.vibe/AGENTS.md` (user-level, lu à chaque session)
- **Gemini** : `~/.gemini/GEMINI.md` (user-level, lu à chaque session)

Contenu recommandé (à adapter) : les 4 règles Karpathy condensées + les 10 règles TDAH condensées. Le skill `i-have-adhd` reste installé pour les sessions où l'on veut le réactiver explicitement (`/i-have-adhd`).

**Vérification** : démarrer une session, demander « résume ce que tu dois faire » — la réponse doit mener par l'action, sans préambule.

## Étape 4 — Installer les 6 commandes slash (Gemini CLI uniquement)

```bash
mkdir -p ~/.gemini/commands
cp -r llm-cli/gemini/commands/* ~/.gemini/commands/
```

Puis dans Gemini CLI : `/commands reload`, puis `/commands list` doit afficher `/plan`, `/debug`, `/review`, `/tdd`, `/brainstorm`, `/discipline`.

**Vérification** : `/plan ajoute une page de contact` doit produire un plan structuré sans toucher au code.

## Étape 5 — Ordre d'adoption recommandé

Ne pas tout activer d'un coup. Semaine par semaine :

1. **Semaine 1** — socle : `karpathy-guidelines`, `verification-gate`, `i-have-adhd` (contexte permanent, étape 3). Rien d'autre.
2. **Semaine 2** — cycle de dev : `tdd`, `systematic-debugging`, `writing-plans`.
3. **Semaine 3** — qualité : `code-review`, `finish-branch`, `work-slicing`.
4. **Semaine 4** — élargissement : `brainstorming`, `prd`, `thought-partner`, `security-audit`, `parallel-agents`.
5. **Semaine 5** — pédagogie & contenu (OpenMAIC) : `feynman-learning`, `deep-research`, `fact-check`, `curriculum-planner`. À activer quand tu produis du contenu éducatif ou documentaire.
6. **Personas Vibe** : activer au besoin, un par un (`code-reviewer` et `debugger` d'abord).

## Étape 6 — Maintenir et faire évoluer

- **Mettre à jour depuis les sources** : surveiller les dépôts amont (superpowers, karpathy-skills et OpenMAIC évoluent vite). Procédure : `git pull` dans un clone frais, relire les `SKILL.md` modifiés, reporter les changements dans `llm-cli/skills/`, re-copier vers `~/.gemini/skills/` et `~/.vibe/skills/`.
- **Ajouter un skill** : suivre la méthode de `writing-skills` (superpowers) — frontmatter `name` + `description` précise, corps concis, test d'activation réel sur les deux CLI avant de valider.
- **Règle de nommage** : kebab-case, nom de dossier = champ `name` du frontmatter.
- **Ne jamais** embarquer de secrets, de chemins absolus personnels, ni de références à des orchestrateurs externes dans les skills de ce dossier.

## Tableau de correspondance final

| Besoin | Gemini CLI | Vibe CLI |
|--------|-----------|----------|
| Règles de comportement de base | `~/.gemini/GEMINI.md` + skill `karpathy-guidelines` | `~/.vibe/AGENTS.md` + skill `karpathy-guidelines` |
| Style de sortie TDAH | skill `i-have-adhd` (+ `GEMINI.md`) | skill `i-have-adhd` (+ `AGENTS.md`) |
| 14 skills de workflow dev | `~/.gemini/skills/` | `~/.vibe/skills/` |
| 4 skills pédagogie & contenu | `~/.gemini/skills/` | `~/.vibe/skills/` |
| Personas spécialisés | commandes `/plan`, `/debug`, `/review`, `/tdd`, `/brainstorm`, `/discipline` | agents `~/.vibe/agents/*.toml` |

## Pièges connus

- **Conflit de contexte** : si `GEMINI.md`/`AGENTS.md` projet contredit un skill, le fichier de contexte gagne souvent. Les skills restent utiles pour les workflows explicites.
- **Trop de skills actifs** : 18 skills = 18 descriptions injectées à chaque session. C'est acceptable, mais si le CLI ralentit au démarrage, désactiver les skills peu utilisés (`disable-model-invocation: true` dans le frontmatter, invocation manuelle uniquement) — candidats : les 4 skills OpenMAIC si tu ne produis pas de contenu éducatif.
- **Oublier `/commands reload`** (Gemini) après chaque modification des TOML.
- **Dossier non trusted** (Vibe) : les skills locaux `.vibe/skills/` ne chargent pas ; préférer le scope user `~/.vibe/skills/`.

## Références

- [Gemini CLI — Skills](https://geminicli.com/docs/cli/skills/) · [Commandes custom](https://geminicli.com/docs/cli/custom-commands/)
- [Mistral Vibe — Skills](https://docs.mistral.ai/vibe/code/cli/skills) · [Agents](https://docs.mistral.ai/vibe/code/cli/agents)
- [Spécification Agent Skills](https://agentskills.io)
