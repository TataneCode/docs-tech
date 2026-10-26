---
name: fact-check
description: "Vérifie les faits d'un contenu existant ou en cours de création : repère les claims à haut risque (chiffres, dates, affirmations absolues, contradictions), les vérifie contre des sources, et produit un rapport court avec preuves. À utiliser pour fiabiliser un cours, un article ou une documentation avant publication."
license: MIT
---

# Fact Check — Vérification factuelle

Garde les erreurs factuelles graves et les hallucinations hors d'un contenu, sans transformer la rédaction en audit exhaustif. Concentre-toi sur les quelques claims qui portent vraiment la confiance du lecteur.

## Deux modes (choisis selon la demande, ne demande pas à l'utilisateur de choisir)

- **Création** : le contenu n'existe pas encore ou on le construit → vérifier **après** sa production, avant livraison.
- **Revue** : le contenu existe déjà et on demande de l'inspecter → rapporter d'abord. Ne modifier que si les corrections ont été demandées ou approuvées après le rapport.

Hors périmètre : la grammaire, le style, la mise en page.

## En création

Produis le contenu normalement. Une fois terminé, relis-le en entier (texte visible + narration). Fais un contrôle de salubrité rapide sur les faits exacts et les contradictions entre sections. Corrige les erreurs évidentes avant livraison — créer le contenu autorise à le rendre factuellement correct. Mentionne brièvement seules les corrections matérielles ou les incertitudes restantes au moment de la livraison. Ne coupe pas la création avec un rapport d'audit séparé, sauf si un claim dépend d'une décision de l'utilisateur.

## En revue d'un contenu existant

1. **Lis le contenu en entier** (texte visible et narration), dans le périmètre demandé.
2. **Lis une fois pour le contexte** et retiens silencieusement une **liste courte de risques à fort signal** :
   - nombres, dates, comptes, noms exacts et citations attribuées ;
   - lois, normes, formules, définitions techniques et classifications ;
   - « premier », « seul », « toujours », « obligatoire » et autres claims absolus ;
   - conclusions causales ou professionnelles présentées comme des faits établis ;
   - contradictions entre sections ;
   - claims très spécifiques sans soutien visible.
3. **Ne vérifie pas tout.** Saute le contenu correct, les préférences de formulation, les simplifications inoffensives et la trivia peu utile.

## Vérifier uniquement la liste courte

- Consulte d'abord les matériels fournis par l'utilisateur : les sources de l'utilisateur peuvent soutenir un claim, mais le contenu vérifié ne peut pas se prouver lui-même.
- Utilise la recherche web pour les claims de la liste courte qui dépendent de connaissances actuelles, exactes, contestées ou spécialisées. Une première passe normale ne devrait pas dépasser **6 à 8 recherches**. Récupère la source pour la lire : un extrait de résultat ou la simple existence d'une source proche n'est pas une preuve.
- Ne mets pas la session en pause pour demander des sources ou des permissions. Utilise les outils disponibles. Si la recherche web est indisponible, continue avec la connaissance stable, fais moins d'engagements factuels, et marque les claims incertains. **Ne devine jamais une URL** : récupère uniquement une URL fournie par l'utilisateur ou retournée par une recherche.
- Pour un énoncé composé, isole la partie douteuse et vérifie exactement cette partie. Privilégie les sources primaires ou officielles. Une source faisant autorité suffit pour une erreur évidente ; ajoute une corroboration seulement pour les claims contestés ou à fort impact. Pour les connaissances versionnées (droit, politique, normes, médecine), vérifie la date, la version et la juridiction pertinentes.
- « Aucune preuve fiable trouvée » ne veut pas dire « faux ». Si la vérification reste inconclusive, dis-le plutôt que d'inventer un verdict ou une correction.

## Préserver les entrées approuvées

Ne fais pas de correction qui entrerait en conflit matériel avec le brief validé, les matériels fournis par l'utilisateur, ou les faits déjà transmis comme entrées de génération. Traite-les comme des entrées approuvées, pas comme du texte généré ordinaire.

Si la preuve indique qu'une entrée approuvée elle-même contient une erreur factuelle : ne modifie pas le contenu concerné et ne surcharge pas silencieusement l'entrée. Signale le conflit à l'utilisateur, indique la section ou le claim concerné et la preuve contraire de façon concise, et propose de garder l'entrée approuvée, d'autoriser la correction factuelle, ou de revoir le conflit sans modifier.

## Le rapport de revue (court et lisible)

En mode revue, retourne environ **3 à 8 findings utiles** à la première passe (ou moins s'il y en a moins). Groupe-les sous ces intitulés, dans cet ordre, et omets un groupe vide :

- **A. Erreur factuelle avérée**
- **B. Formulation trompeuse**
- **C. Vérification nécessaire** — uniquement quand le claim compte

Numérote les findings en continu dans tout le rapport. Chaque finding a une ligne en gras courte avec son numéro, sa localisation et son problème, par exemple : `**1. Section 5, « quiz sur les réunions » : confusion sur les seuils**`.

Sous chaque intitulé, exactement trois puces :

- **Énoncé original :** cite uniquement la phrase ou le fragment concerné ;
- **Problème constaté :** explique l'erreur et le fait correct en langage clair ; indique une source concise et sa date quand utile ;
- **Correction proposée :** donne uniquement l'action d'édition ou un remplacement compact.

Une puce par idée, courte. Ne répète pas le même fait ou la même citation d'une puce à l'autre. Ne montre pas de scores, de pourcentages de confiance, de méthodologie interminable, de claims corrects ou de problèmes de style mineurs. Ne gonfle pas le rapport pour atteindre un quota. Si aucun problème matériel n'est trouvé, dis quel périmètre a été parcouru et qu'aucune erreur évidente n'a été trouvée ; ne prétends pas que le contenu est parfaitement exact.

## Laisser l'utilisateur choisir après une revue

Quand il y a des findings actionnables et que les corrections n'ont pas déjà été autorisées, la dernière action du tour propose à l'utilisateur de choisir entre :

- corriger tous les problèmes rapportés ;
- corriger uniquement les erreurs confirmées ;
- garder le rapport sans modification.

Ne modifie rien avant la réponse. Après approbation, relis chaque section sélectionnée et ne change que les claims approuvés.

## Vérification avant de terminer

- [ ] Seuls les claims à fort signal ont été vérifiés, pas tout le contenu.
- [ ] Chaque finding a une localisation, une preuve et une correction proposée.
- [ ] Les entrées approuvées (brief, matériels, faits transmis) n'ont pas été contredites sans signalement.
- [ ] L'inconclus est dit comme tel, jamais présenté comme une certitude.
