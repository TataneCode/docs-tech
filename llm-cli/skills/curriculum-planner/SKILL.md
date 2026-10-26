---
name: curriculum-planner
description: "Planifie et construit une série de plusieurs leçons cohérentes (ex. « 7 jours pour apprendre Python », un parcours d'onboarding, un semestre découpé en séances) : clarifie le brief en plusieurs tours, fait valider la liste complète des leçons avant de produire quoi que ce soit, puis construit les leçons une par une."
license: MIT
---

# Curriculum Planner — Planification d'une série de leçons

Une série, c'est plusieurs leçons que l'apprenant suit dans l'ordre. L'unité de travail reste **une leçon à la fois**. Trois responsabilités à ce niveau uniquement : obtenir un brief clair, obtenir la **validation explicite de l'utilisateur sur la liste complète des leçons** avant de produire quoi que ce soit, et porter la série du début à la fin sans perdre le fil.

## Quand l'utiliser

- L'utilisateur demande une séquence de plusieurs leçons : « 7 jours pour apprendre Python », un parcours d'onboarding de 4 semaines, un semestre découpé en séances, « transformer ce livre en une leçon par chapitre ».

## Quand ne pas l'utiliser

- Pour une seule leçon, aussi grosse soit-elle → utiliser le skill de conception de leçon (`writing-plans` pour le plan, `brainstorming` pour le cadrage).

## Porte 1 — Clarifier, en plusieurs tours

Une demande de série est presque toujours sous-déterminée. « 7 jours pour apprendre Python » fixe le sujet et le nombre de leçons, et laisse ouvert tout ce qui décide de leur contenu. Travaille en **2 ou 3 tours**, un par un, chaque tour plus étroit que le précédent.

**Tour 1 — où ça atterrit.** Les 2–3 choses que rien d'autre ne peut décider sans :

- **qui est l'apprenant** — débutant absolu, quelqu'un qui code dans un autre langage, une équipe de niveaux mixtes ;
- **ce qu'il veut en retirer** — un script fonctionnel à lui, un examen à passer, un concept à pouvoir présenter.

Mets l'exemple dans le libellé de l'option : « débutant complet, jamais écrit une ligne de code » dit ce que tu entends par débutant.

**Tour 2 — la forme, ajustée aux réponses du tour 1.** Maintenant pose ce que le tour 1 rend posable : la durée d'une séance, le rythme, la langue d'enseignement, le degré de pratique. **Construis les options à partir de ce qu'il vient de répondre** : « débutant complet » devient « format court : manipuler un petit outil / atelier autonome / mini-projet guidé », et « écrit déjà du Java » devient « couper sur les différences Python/Java, ou repartir de zéro ».

**Tour 3 — les bords, et seulement s'ils sont réels.** S'il dispose d'un livre, d'un syllabus ou d'un support existant à partir duquel construire la série ; combien de vérifications il veut (un quiz par leçon, un seul à la fin, aucun). Saute ce tour quand aucune des deux questions ne changerait le plan.

Règles pour tous les tours :

- **Au plus 3 grands sujets par tour.** Un tour qui porte six questions, c'est le formulaire géant qui se déguise.
- **Ouvre chaque tour en disant ce que tu retiens du précédent.** « Vu que c'est débutant complet, 30 minutes par leçon : je compresse chaque leçon sur un outil exécutable. » L'utilisateur doit voir sa réponse être utilisée. C'est toute la raison pour laquelle plusieurs tours passent pour professionnels plutôt que lents.
- **Ne demande jamais ce que tu sais déjà.** Ce que la demande a tranché, ou ce que tu peux raisonnablement prendre par défaut, est **énoncé comme ta décision, que l'utilisateur peut corriger** plutôt que posé en question. « Par défaut en français, 30 minutes par leçon, en direct » ne coûte aucun tour.

## Porte 2 — La porte de confirmation

Avant de produire la moindre leçon, présente la **liste complète des leçons** de la série :

```markdown
# Série : [titre]

## Brief retenu
- Public : [...]
- Objectif : [...]
- Contraintes : [...]

## Liste des leçons
| # | Titre | Objectif | Vérification |
|---|-------|----------|--------------|
| 1 | [...] | [ce que l'apprenant saura faire] | [exercice/quiz] |
| 2 | [...] | [...] | [...] |

## Ordre et dépendances
La leçon N s'appuie sur la leçon M...

## Hors scope
[...]
```

**Attends la validation explicite.** L'utilisateur valide, corrige, ou réduit. Aucune production avant cette validation — c'est ce qui distingue un plan de série d'une accumulation de leçons.

## Concevoir la série

- La colonne vertébrale est un **petit nombre de concepts** qui reviennent, chacun à un niveau de complexité, d'abstraction ou de transfert croissant. Ce n'est pas une liste linéaire de sujets.
- Chaque leçon a : un objectif apprenant, un contenu, une pratique guidée, une vérification.
- Chaque leçon produit quelque chose de vérifiable (exercice, quiz, livrable).
- La progression est lisible : l'apprenant doit pouvoir dire où il en est et ce qui vient ensuite.

## La boucle d'exécution

Pour chaque leçon, dans l'ordre :

1. **Conçois** la leçon (objectif, contenu, pratique, vérification).
2. **Produis** la leçon (un livrable concret : fiche, atelier, exercice, quiz).
3. **Vérifie** : la vérification de la leçon est exécutée ou résoluble, et le résultat est consigné.
4. **Passe à la suivante**, en rappelant brièvement le fil conducteur.

Discipline de contexte long :

- Ne retravaille pas une leçon terminée sauf demande explicite.
- À chaque nouvelle leçon, rappelle en une ligne la leçon précédente et ce qu'elle a produit.
- Si une leçon échoue ou bloque, dis-le, marque-la, et demande comment continuer plutôt que d'improviser seul.

## Pour terminer

Quand la dernière leçon est produite et vérifiée :

- Récapitule la série : leçons produites, vérifications exécutées (commandes + résultats si code), ordre suivi.
- Liste ce qui reste hors scope.
- Propose la suite logique (révision, approfondissement, leçon suivante hors série).

## Vérification avant de terminer

- [ ] Le brief a été clarifié en tours (pas un formulaire géant).
- [ ] La liste complète des leçons a été **validée explicitement** avant production.
- [ ] Chaque leçon a objectif + contenu + pratique + vérification.
- [ ] Chaque leçon a été produite et sa vérification exécutée/consignée.
- [ ] Le fil conducteur est resté visible d'une leçon à l'autre.
- [ ] Le hors scope est stated à la fin.
