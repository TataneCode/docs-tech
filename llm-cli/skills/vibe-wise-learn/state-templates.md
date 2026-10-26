# Mod\u00e8les d'\u00e9tat local

Cr\u00e9e uniquement ces trois fichiers dans le `.vibe-wise/` du projet choisi, ou
son ancien `.sensible-vibes/` en cas de reprise. Utilise le r\u00e9pertoire s\u00e9lectionn\u00e9
par SKILL.md. Remplace les valeurs entre crochets par des preuves r\u00e9elles ou
\u00ab Not specified \u00bb (Non renseign\u00e9). Garde les deux lignes de statut non
format\u00e9es et proches du haut. Ne remplace pas un \u00e9tat existant par un mod\u00e8le
neuf.

## profile.md

```markdown
# Profil de l'apprenant

Learning mode: active
Onboarding: complete

## Projet
Situation: [New / Existing / Known]
Building: [objectif du projet]
Codebase familiarity: [r\u00e9ponse]
Learning scope: [entire system / parts we touch / mixed]

## Exp\u00e9rience
Overall programming: [Beginner / Intermediate / Advanced, ou Not specified]
Stack familiarity: [niveaux par technologie si donn\u00e9s ; sinon Not specified]

## Objectifs
Primary: [r\u00e9ponse]
Capability goal: [r\u00e9ponse optionnelle]

## Pr\u00e9f\u00e9rences
Checkpoint frequency: Normal
Question style: Open-ended
Implementation style: AI writes code

## Strong Concepts
No demonstrated understanding recorded yet.

## Developing Concepts
None recorded yet.

## Revisit
None recorded yet.
```

## progress.md

```markdown
# Progr\u00e8s d'apprentissage

No learning events recorded yet.
```

Au fil de l'apprentissage, ajoute un `## Sujet` avec des puces concises sous
Introduced (Introduit), Demonstrated understanding (Compr\u00e9hension d\u00e9montr\u00e9e) et
Needs reinforcement (\u00c0 renforcer). Enregistre des preuves de raisonnement, pas
des citations d'un \u00e9change entier. Les pr\u00e9f\u00e9rences de produit \u00e9tablissent des
besoins ; ce ne sont pas des preuves de compr\u00e9hension d'ing\u00e9nierie. Garde le
raisonnement propos\u00e9 par l'apprenant distinct des concepts expliqu\u00e9s par l'IA.
Consolide les entr\u00e9es r\u00e9p\u00e9t\u00e9es. Garde chaque sujet lisible ind\u00e9pendamment pour
qu'il puisse \u00eatre charg\u00e9 sans tout le fichier. En attendant un checkpoint, garde une
courte section intitul\u00e9e exactement `## Pending decision` (D\u00e9cision en attente), hors
d'un sujet, avec l'approche propos\u00e9e et la r\u00e9ponse attendue. Inclus le nom de la
d\u00e9cision et l'\u00e9tape du checkpoint : en attente de raisonnement, de confirmation du
choix, ou d'approbation d'impl\u00e9mentation. Supprime-la une fois r\u00e9solue : enregistre
le r\u00e9sultat dans la carte comme choisi ou impl\u00e9ment\u00e9, en indiquant les ajouts
approuv\u00e9s comme \u00e9tant de l'IA, et toute preuve de raisonnement dans son sujet. Ne
d\u00e9place pas le texte en attente dans un sujet ; mets \u00e0 jour les entr\u00e9es que le
r\u00e9sultat remplace. Enregistre les choix confirm\u00e9s dans la carte sans pr\u00e9tendre
qu'ils sont impl\u00e9ment\u00e9s. Garde tout p\u00e9rim\u00e8tre de codage propos\u00e9 explicite. La
confirmation ne couvre que la proposition pr\u00e9sent\u00e9e. N'ajoute pas de champs, de
comportements, d'alternatives rejet\u00e9es ou de raisons non mentionn\u00e9s au design
choisi. Marque les d\u00e9tails non r\u00e9solus comme inconnus et les suggestions de l'IA
comme propos\u00e9es ; ne les attribue jamais \u00e0 l'apprenant.

## project-map.md

```markdown
# Carte du projet

## Objectif
[Ce que fait ce logiciel.]

## Besoins
[Besoins et contraintes de l'utilisateur. Ils ne d\u00e9cident pas automatiquement des choix techniques.]

## Composants
[Composants, responsabilit\u00e9s et chemins de fichiers associ\u00e9s.]

## Flux principal
[Diagramme texte compact avec fl\u00e8ches \u00e9tiquet\u00e9es. Marque les inconnus et distingue composants propos\u00e9s, choisis et impl\u00e9ment\u00e9s. Refl\u00e8te le mod\u00e8le de l'apprenant affin\u00e9 ensemble, ou du code existant v\u00e9rifi\u00e9 ; ne comble pas les relations manquantes par des designs suppos\u00e9s.]

## Donn\u00e9es et fronti\u00e8res de confiance
[Stockage, propri\u00e9t\u00e9, auth, services externes ; inconnu si non v\u00e9rifi\u00e9.]

## Build et d\u00e9ploiement
[Commandes et chemins de configuration v\u00e9rifi\u00e9s dans le d\u00e9p\u00f4t.]

## Inconnus
[Choix techniques non r\u00e9solus et ce qui n\u00e9cessite une inspection, y compris stack, emplacement/mod\u00e8le de stockage, structures de donn\u00e9es, interfaces et choix de d\u00e9ploiement.]
```
