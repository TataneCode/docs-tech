# Synthèse des sources — skills, agents et commandes

Objectif : extraire de quatre dépôts publics ce qui est utile pour **Gemini CLI** et **Vibe CLI / Vibe Code**, sans dépendre de Claude Code ni d'aucune orchestration multi-fournisseurs.

## Les 6 dépôts étudiés

| Dépôt | Contenu | Ce qu'on en tire |
|-------|---------|------------------|
| [obra/superpowers](https://github.com/obra/superpowers) | 14 skills de discipline (TDD, debug, plans, revue, worktrees...) + scripts shell | Le squelette complet du cycle de développement agentique, 100 % portable |
| [multica-ai/andrej-karpathy-skills](https://github.com/multica-ai/andrej-karpathy-skills) | 1 skill `karpathy-guidelines` (MIT) + fichiers `CLAUDE.md`/`CURSOR.md` équivalents | Règles comportementales de base : simplicité, chirurgical, vérifier |
| [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd) | 1 skill `i-have-adhd` (MIT) + hooks toujours-actifs | Règles de formatage de sortie pour un lecteur TDAH |
| [nyldn/claude-octopus](https://github.com/nyldn/claude-octopus) | 63 skills, 54 commandes slash, 31 personas, orchestrateur multi-LLM (`bin/octopus`) | Des workflows mûrs (slicing, PRD, thought-partner, audit sécurité), mais très couplés à Claude Octopus |
| [THU-MAIC/OpenMAIC](https://github.com/THU-MAIC/OpenMAIC) | 25 skills de pédagogie agentique (MIT) | 4 skills de pédagogie retenus, portés en version générique |
| [nykooi1/vibe-wise](https://github.com/nykooi1/vibe-wise) | Plugin apprentissage pour Claude Code + Codex : skills `learn`/`reset`, état local `.vibe-wise/`, hook SessionStart, script Python de reset | Un **mode apprentissage complet** : l'apprenant conduit le design, l'IA écrit le code retenu. État local portable, checkpoints structurés, onboarding guidé |

## Ce qui est retenu, et pourquoi

### 1. obra/superpowers — colonne vertébrale (14 skills, tous portables)

Le dépôt définit un cycle de vie complet de la tâche agentique. Chaque skill est un `SKILL.md` au standard Agent Skills, sans dépendance externe (les scripts shell référencés sont optionnels). Ce sont les **fondations** :

- `verification-before-completion` — **loi de fer** : aucune affirmation de succès sans commande exécutée et sortie constatée. C'est le garde-fou le plus important.
- `systematic-debugging` — 4 phases : cause racine → analyse de pattern → hypothèse testée → correctif. Interdit de corriger avant de comprendre.
- `test-driven-development` — Red-Green-Refactor avec vérification explicite du RED puis du GREEN.
- `writing-plans` — plan en tâches numérotées, chaque étape avec sa commande de vérification.
- `brainstorming` — cadrer le besoin réel avant toute solution (questions, rejeu, sizing).
- `requesting-code-review` / `receiving-code-review` — revue proactive avant merge, et réception rigoureuse des retours (pas d'acquiescement performatif).
- `finishing-a-development-branch` — tests, merge local ou PR, nettoyage du worktree.
- `using-git-worktrees` — isolation des travaux (peut être fusionné dans `finish-branch` pour nos CLI).
- `dispatching-parallel-agents` — patron de dispatch parallèle (utile si le CLI hôte a des sous-agents).
- `subagent-driven-development` / `executing-plans` — exécution de plan avec sous-agents ou en ligne.
- `using-superpowers` — méta-règle : invoquer un skill avant toute réponse.
- `writing-skills` — méthode pour écrire/tester ses propres skills (utile pour la suite).
- `diagnosing-superpowers` — spécifique au dépôt, **non retenu**.

### 2. karpathy-guidelines — socle comportemental (1 skill)

4 règles courte et denses, complémentaires de superpowers :

1. **Réfléchir avant de coder** — expliciter les hypothèses, dire quand c'est flou.
2. **Simplicité d'abord** — code minimum, rien de spéculatif.
3. **Changements chirurgicaux** — ne toucher que ce qui est demandé, nettoyer seulement ses propres orphelins.
4. **Exécution guidée par les objectifs** — critère de succès vérifiable, boucle jusqu'à vérification.

Le dépôt fournit aussi `CLAUDE.md` et `CURSOR.md` : pour Gemini/Vibe, le skill suffit (Gemini lit `GEMINI.md`, Vibe lit `AGENTS.md` — voir [02-formats-cibles.md](02-formats-cibles.md)).

### 3. i-have-adhd — style de sortie (1 skill)

10 règles de formatage pensées pour un cerveau TDAH, applicables à n'importe quel agent :

1. Mener par **l'action suivante** (première ligne = action).
2. **Numéroter** les tâches multi-étapes.
3. Terminer par **une seule prochaine action** concrète (< 2 min).
4. **Supprimer les tangentes** (une seule question à la fin).
5. **Restater l'état** à chaque tour (étape N sur M).
6. **Estimations de temps concrètes** (15 min / un après-midi, pas « un peu de travail »).
7. **Rendre les gains visibles** (ce qui marche maintenant, testable).
8. Ton factuel sur les erreurs (cause + fix, pas « oups »).
9. Limiter les listes à 5 items visibles.
10. **Pas de préambule, pas de récapitulatif, pas de formules de politesse.**

Le dépôt embarque aussi des hooks « always-on » (sh/ps1/mjs) pour forcer l'injection à chaque session — nous retenons le principe sous forme de skill, pas de hook système.

### 4. claude-octopus — workflows avancés, à dé-claudifier

Le dépôt est puissant mais **couplé à son orchestrateur** (`scripts/orchestrate.sh`, 12 fournisseurs externes, consensus à 75 %). On ne garde que les skills dont la logique interne est autonome, en retirant :

- les appels à `orchestrate.sh`, `${CLAUDE_PLUGIN_ROOT}`, `~/.claude-octopus/plugin/...`
- les mentions de fournisseurs externes (Codex, Antigravity, Grok...)
- les bannières « CLAUDE OCTOPUS ACTIVATED » et le vocabulaire `/octo:*`
- les références au tracker `bd` (beads) dans `work-slicing` → remplacé par une sortie liste de tâches

Skills retenus de cet dépôt :

| Skill octopus | Retenu sous le nom | Adaptation |
|---------------|--------------------|------------|
| `skill-verification-gate` | `verification-gate` | Idée fusionnée avec `verification-before-completion` de superpowers (doublon sémantique) |
| `skill-tdd` | `tdd` | Fusionné avec `test-driven-development` (red-green avec preuve observée) |
| `skill-writing-plans` | `writing-plans` | Fusionné avec `writing-plans` de superpowers |
| `skill-finish-branch` | `finish-branch` | Étape « revue multi-fournisseurs » remplacée par une revue locale standard |
| `skill-work-slicing` | `work-slicing` | Sortie `bd` remplacée par une liste de tranches + dépendances |
| `skill-prd` | `prd` | Phases clarification → recherche (max 2 recherches) → PRD → auto-évaluation |
| `skill-thought-partner` | `thought-partner` | 4 techniques de percée (pattern spotting, paradox hunting, naming the unnamed, contrast creation) |
| `skill-debug` | `systematic-debugging` | Renforcé par les 4 phases de superpowers (choix : garder la version superpowers, plus complète) |
| `skill-audit` / `octopus-security-audit` | `security-audit` | Version mono-agent : scan OWASP, secrets, dépendances, revue de config |
| `skill-parallel-agents` | `parallel-agents` | Patron de dispatch, adapté aux sous-agents Vibe/Gemini si disponibles |
| `skill-code-review` | `code-review` | Fusionné avec `requesting/receiving-code-review` de superpowers |
| `skill-authoring` | — (méthode) | Conservé comme référence dans [03-plan-preparation.md](03-plan-preparation.md), pas comme skill installé |

**Non retenus** (trop couplés à l'orchestrateur multi-LLM ou hors sujet) : `flow-*` (Discover/Define/Develop/Deliver/Parallel/Spec), `skill-council`, `skill-debate`, `skill-decision-support`, `skill-pressure-test`, `skill-doc-*`, `skill-extract`, `skill-content-pipeline`, `skill-ui-ux-design`, `skill-deck`, `skill-doctor`, `skill-issues`, `skill-resume`, `skill-status`, `skill-rollback`, `skill-iterative-loop`, `skill-factory`, `skill-coverage-audit`, `skill-cost-projections`, `skill-agent-topology`, `skill-context-detection`, `skill-copilot-provider`, `skill-native-escalation-routing`, `skill-staged-review`, `skill-quick`, `skill-meta-prompt`, `skill-prototype`, `skill-design-lineage`, `skill-knowledge-work`, `skill-review-response`, `skill-task-management`, `skill-verify`, `skill-visual-feedback`, `sys-configure`, et les 54 commandes `/octo:*` (elles ne sont que des coquilles qui délèguent à l'orchestrateur).

### Personas retenus (claude-octopus → agents Vibe)

Le dépôt définit 31 personas. On retient les 7 profils les plus utiles en mono-agent, traduits au format TOML de Vibe (`~/.vibe/agents/*.toml` + prompt dans `~/.vibe/prompts/`) :

| Persona source | Agent Vibe | Rôle |
|----------------|-----------|------|
| `code-reviewer` | `code-reviewer` | Revue de code : qualité, sécurité, performance |
| `debugger` | `debugger` | Spécialiste diagnostic : stack traces, logs, cause racine |
| `security-auditor` | `security-auditor` | Audit OWASP, menaces, conformité |
| `backend-architect` | `backend-architect` | Design d'API, microservices, résilience |
| `docs-architect` | `docs-architect` | Documentation technique long format à partir du code |
| `tdd-orchestrator` | `tdd-orchestrator` | Discipline red-green-refactor |
| `performance-engineer` | `performance-engineer` | Profilage, N+1, caching, Core Web Vitals |

### Commandes retenues (claude-octopus → commandes Gemini CLI)

Gemini CLI n'a pas de mécanisme de personas ; les commandes slash TOML remplacent les entrées `/octo:*` pour les 6 workflows les plus fréquents : `/plan`, `/debug`, `/review`, `/tdd`, `/brainstorm`, `/discipline` (mode rigueur on/off). Voir `gemini/commands/`.

### 5. THU-MAIC/OpenMAIC — pédagogie agentique (25 skills, 4 retenus)

OpenMAIC est une plateforme de « classroom » multi-agents. Son répertoire `skills/agent-runtime/` contient 25 skills de **conception pédagogique** très mûres, mais exprimées pour le runtime OpenMAIC (outils `create_stage`, `generate_scene`, `set_roster`, DSL de slides, scènes `interactive`/`slide`/`quiz`). La plupart ne sont pas portables telles quelles ; la **méthode** l'est.

Retenus (méthode extraite, OpenMAIC retiré, généralisée à tout contenu pédagogique) :

| Skill source | Retenu sous le nom | Ce qui est gardé |
|--------------|--------------------|-----------------|
| `feynman-learning` | `feynman-learning` | Le cycle en 8 étapes : teach-back d'abord, un seul trou à la fois, questions socratiques avant réponses, chasse au jargon, stress-test des analogies, transfert, enregistrement final |
| `deep-research` | `deep-research` | Recherche avant rédaction, facettes, budget (max 8 recherches / 6 URLs), registre claim → source, règles dures (jamais d'URL inventée, conflits présentés) |
| `fact-check` | `fact-check` | Revue ciblée (pas d'audit exhaustif), liste courte de claims à risque, rapport en 3 groupes (erreur avérée / formulation trompeuse / vérification nécessaire), choix utilisateur après revue |
| `curriculum-planner` | `curriculum-planner` | Clarification en 2–3 tours, porte de confirmation sur la liste complète des leçons avant production, boucle leçon par leçon |

**Non retenus** (trop couplés au runtime OpenMAIC ou redondants avec nos skills existants) : `stage-design`, `stage-dsl`, `slide-dsl`, `slide-craft`, `style-clone`, `page-clone`, `teacher-style-clone`, `pptx-import`, `pro-editing` (outils OpenMAIC), `lecture-style`, `workshop-style`, `deep-interactive`, `vocational`, `k12-core-literacy-planning` (formats de cours OpenMAIC), `learning-to-learn`, `social-emotional-learning`, `understanding-by-design`, `spiral-curriculum`, `zone-of-proximal-development` (méthodes pédagogiques valides mais exprimées en DSL OpenMAIC — à re-porter au cas par cas si besoin), `build-personal-skill` (méta, spécifique OpenClaw), et le skill `openmaic` (setup de la plateforme elle-même, hors scope).

### 6. nykooi1/vibe-wise — mode apprentissage (2 skills)

Le dépôt définit un plugin **Claude Code + Codex** qui met l'apprentissage en premier :
l'IA demande l'approche de l'apprenant, l'aide à examiner les compromis, explique les
concepts méconnus, puis écrit le code retenu après confirmation explicite. L'apprenant
reste propriétaire des décisions de design.

Éléments retenus :

| Élément source | Retenu sous le nom | Adaptation |
|----------------|--------------------|------------|
| `skills/learn/` (SKILL.md + behavior.md + onboarding.md + state-templates.md) | `vibe-wise-learn` | Traduit en français, `AskUserQuestion` → question texte simple, suppression du hook SessionStart (pas d'équivalent Vibe/Gemini) |
| `skills/reset/` (SKILL.md + reset.py) | `vibe-wise-reset` | Traduit en français, `reset.py` rendu autonome (lookup d'état intégré, plus d'import depuis `hooks/session_start.py`) |
| État local `.vibe-wise/` (profile.md, progress.md, project-map.md) | conservé tel quel | Compatible entre Claude Code, Codex, Vibe et Gemini CLI — les notes sont interchangeables |

**Non retenus** : le hook `SessionStart` de Claude Code (Vibe et Gemini n'ont pas d'équivalent
— l'utilisateur invoque `/vibe-wise-learn` manuellement à chaque session), les références
`${CLAUDE_PLUGIN_ROOT}`, le picker natif `AskUserQuestion`, et les fichiers spécifiques
Codex (`codex.md`, `agents/openai.yaml`).

## Synthèse : ce que chaque CLI reçoit

| Élément | Gemini CLI | Vibe CLI / Vibe Code |
|---------|-----------|----------------------|
| 20 skills `SKILL.md` | ✅ `~/.gemini/skills/` (activation auto via `description`) | ✅ `~/.vibe/skills/` (slash command `/nom` + activation auto) |
| Personas | ❌ (pas de mécanisme) → remplacés par 8 commandes TOML `~/.gemini/commands/` | ✅ 7 agents TOML `~/.vibe/agents/` + prompts `~/.vibe/prompts/` |
| Contexte permanent (style Karpathy + TDAH) | `GEMINI.md` à créer (voir plan) | `AGENTS.md` à créer (voir plan) |

## Pièges connus

- **Doublons sémantiques** : superpowers et octopus ont chacun leur skill « verification », « tdd », « plans », « review », « debug ». Ne pas installer les deux versions : ce dépôt fusionne chaque paire en un seul skill.
- **Skills trop bavards** : les SKILL.md originaux d'octopus font 100–650 lignes avec des contrats d'exécution verbeux. Les versions portées ici sont condensées pour rester dans le budget de contexte d'un skill.
- **Activation** : Gemini et Vibe n'activent un skill que si la `description` du frontmatter correspond à la tâche. Une description trop générique = skill jamais invoqué ; trop étroite = idem. Les descriptions portées ici gardent le vocabulaire déclencheur des sources.
- **Hooks i-have-adhd** : les hooks `always-on.*` du dépôt original sont spécifiques à certains hôtes ; ne pas les installer tels quels.

## Références

- [Superpowers (obra)](https://github.com/obra/superpowers) — [docs](https://github.com/obra/superpowers/tree/main/docs)
- [andrej-karpathy-skills (multica-ai)](https://github.com/multica-ai/andrej-karpathy-skills)
- [i-have-adhd (ayghri)](https://github.com/ayghri/i-have-adhd)
- [claude-octopus (nyldn)](https://github.com/nyldn/claude-octopus)
- [OpenMAIC (THU-MAIC)](https://github.com/THU-MAIC/OpenMAIC) — répertoire `skills/agent-runtime/`
- [vibe-wise (nykooi1)](https://github.com/nykooi1/vibe-wise) — plugin apprentissage (MIT)
- [Spécification Agent Skills (agentskills.io)](https://agentskills.io)
- [Documentation Gemini CLI — Skills](https://geminicli.com/docs/cli/skills/)
- [Documentation Gemini CLI — Commandes custom](https://geminicli.com/docs/cli/custom-commands/)
- [Documentation Mistral Vibe — Skills](https://docs.mistral.ai/vibe/code/cli/skills)
- [Documentation Mistral Vibe — Agents](https://docs.mistral.ai/vibe/code/cli/agents)
