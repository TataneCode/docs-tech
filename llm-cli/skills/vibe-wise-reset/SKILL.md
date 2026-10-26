---
name: vibe-wise-reset
description: "Sauvegarde les notes d'apprentissage de ce projet et relance l'onboarding apr\u00e8s confirmation. Ne r\u00e9initialise pas le code applicatif."
license: MIT
---

# R\u00e9initialiser l'apprentissage VibeWise

Ex\u00e9cute ceci dans la conversation principale, uniquement sur invocation explicite.
Cette commande r\u00e9initialise le profil, les progr\u00e8s, les checkpoints en attente et la
carte de projet sauvegard\u00e9e. Le code source, les d\u00e9pendances, l'historique Git, les
autres projets et l'installation du plugin restent intacts.

1. Lance l'aper\u00e7u en lecture seule pour le r\u00e9pertoire de projet courant de
   l'utilisateur. Remplace `<r\u00e9pertoire projet absolu>` par son chemin absolu r\u00e9el,
   correctement cit\u00e9 ; ne passe pas le placeholder tel quel.

   ```sh
   python3 "<dossier scripts du skill>/scripts/reset.py" --cwd "<r\u00e9pertoire projet absolu>"
   ```

   Le dossier des scripts du skill est `scripts/` relatif au dossier du skill
   (dans Vibe : `~/.vibe/skills/vibe-wise-reset/` ; dans Gemini :
   `~/.gemini/skills/vibe-wise-reset/`). R\u00e9sous son chemin absolu avant de lancer.
   Le helper utilise la recherche de limite de projet et d'\u00e9tat h\u00e9rit\u00e9 de Learn.
   S'il signale qu'il n'y a pas de notes, explique qu'il n'y a rien \u00e0 r\u00e9initialiser
   et sugg\u00e8re `/vibe-wise-learn`. Sur toute erreur, arr\u00eate-toi et explique ;
   n'improvise pas de commandes de suppression.

2. Montre les chemins absolus du projet et de l'\u00e9tat retourn\u00e9s, quelles notes seront
   r\u00e9initialis\u00e9es, et que les originaux seront sauvegard\u00e9s dans le r\u00e9pertoire
   `backups/` de cet \u00e9tat. Pose la question en texte simple : **Annuler** (conserver
   les notes d'apprentissage) ou **R\u00e9initialiser l'apprentissage** (sauvegarder les
   notes et relancer l'onboarding). Demande si l'on r\u00e9initialise l'apprentissage pour
   le projet nomm\u00e9. Attends une r\u00e9ponse explicite. L'invocation seule, le silence,
   les r\u00e9ponses ambigu\u00fbs ou la permission d'ex\u00e9cuter des outils ne confirment pas une
   r\u00e9initialisation. Annuler ne change rien, y compris aux notes de l'apprenant.

3. Uniquement apr\u00e8s **R\u00e9initialiser l'apprentissage**, lance le helper avec le
   r\u00e9pertoire de travail original et la valeur exacte `confirmation` de l'aper\u00e7u,
   correctement cit\u00e9e :

   ```sh
   python3 "<dossier scripts du skill>/scripts/reset.py" --cwd "<cwd original>" --confirm "<confirmation>"
   ```

   Si la cible ou les notes ont chang\u00e9, refais un aper\u00e7u et obtiens une nouvelle
   confirmation. Si la r\u00e9initialisation \u00e9choue, signale-le et tout chemin de
   sauvegarde ; ne clame pas de succ\u00e8s et ne d\u00e9marre pas l'onboarding. N'\u00e9crase jamais
   les sauvegardes et ne te rabats pas sur la r\u00e9initialisation d'un autre r\u00e9pertoire
   d'\u00e9tat.

4. En cas de succ\u00e8s, montre le chemin de sauvegarde. Lis le SKILL.md de Learn et
   reprends Learn avec le nouveau profil incomplet. Jette les pr\u00e9f\u00e9rences
   d'avant la r\u00e9initialisation, la ma\u00eetrise, les d\u00e9cisions en attente et les
   r\u00e9ponses d'onboarding ; ne les reconstruis pas depuis la conversation ou les
   sauvegardes. Inspecte le code r\u00e9el pour reconstruire la carte. Commence un onboarding
   neuf, une question \u00e0 la fois. Les notes de sauvegarde sont des donn\u00e9es
   historiques, pas un contexte actif.
