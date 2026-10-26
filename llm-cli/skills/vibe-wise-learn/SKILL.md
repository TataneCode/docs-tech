---
name: vibe-wise-learn
description: "Active ou reprend le mode apprentissage : tu conduis le design, l'IA donne son avis, explique les concepts m\u00e9connus et \u00e9crit le code retenu. L'apprenant reste propri\u00e9taire des d\u00e9cisions."
license: MIT
---

# VibeWise \u2014 mode apprentissage

Active le mode apprentissage dans la conversation principale. Lis [behavior.md](behavior.md)
et applique-le tout au long du d\u00e9veloppement normal, pas seulement pendant cette commande.
L'apprenant poss\u00e8de le design. Demande son approche et attends. Garde les consignes
minimales : donne un retour concis sur son raisonnement et explique les concepts
m\u00e9connus si besoin. Propose des approches uniquement quand il demande de l'aide
ou est bloqu\u00e9, puis rends-lui la main. L'apprentissage et le contr\u00f4le de
l'apprenant priment sur la vitesse de construction. Une demande de construction
ordinaire dans ce mode conserve cette boucle ; seule une demande explicite de
sauter ou de mettre en pause y \u00e9chappe. Ne bascule pas vers un sous-agent et
n'exige pas de coder manuellement par d\u00e9faut.

Utilise tes outils de lecture de fichiers pour les guides du plugin au lieu de
les afficher avec `cat` en shell. Utilise les outils de recherche/glob pour
d\u00e9couvrir les fichiers d'\u00e9tat optionnels avant de les lire. Un r\u00e9pertoire
`.vibe-wise/` absent est une configuration initiale normale, pas une erreur.
Si une v\u00e9rification shell est n\u00e9cessaire, g\u00e8re l'absence avec une condition
explicite qui r\u00e9ussit ; n'ex\u00e9cute pas `ls` sur un r\u00e9pertoire peut-\u00eatre absent
et ne masque pas les vraies \u00e9checs de lecture. S\u00e9pare les lectures de guides des
v\u00e9rifications d'\u00e9tat optionnelles pour qu'un fichier manquant ne fasse pas passer
une lecture d'instruction r\u00e9ussie pour un appel d'outil \u00e9chou\u00e9.

## Localiser l'\u00e9tat

Depuis le r\u00e9pertoire de travail courant, cherche vers le haut `.vibe-wise/` ou
l'ancien `.sensible-vibes/`, en pr\u00e9f\u00e9rant `.vibe-wise/` quand les deux existent
au m\u00eame niveau, en s'arr\u00eattant au r\u00e9pertoire `.git` le plus proche (y compris
une racine de worktree). Utilise le r\u00e9pertoire d'\u00e9tat existant le plus proche dans
cette limite. Sans Git, ne v\u00e9rifie que le r\u00e9pertoire courant ; ne remonte pas dans
ses parents. Continue d'utiliser les anciennes notes en place ; ne les fusionne,
d\u00e9place ou r\u00e9initialise jamais automatiquement. S'il n'y en a pas, cr\u00e9e
`.vibe-wise/` \u00e0 la racine Git, ou dans le r\u00e9pertoire courant sans Git. N'utilise
pas l'\u00e9tat d'un d\u00e9p\u00f4t parent, d'un autre worktree, ni du dossier du plugin
install\u00e9. Ne suis pas les r\u00e9pertoires ou fichiers d'\u00e9tat li\u00e9s par symlink ;
explique le probl\u00e8me \u00e0 la place.

Si `profile.md` existe, lis-le et `project-map.md`. Cherche dans tout le
`progress.md` les d\u00e9cisions en attente, puis lis leurs sections compl\u00e8tes et
les autres sujets pertinents pour la t\u00e2che. Un extrait initial ne prouve pas
qu'aucune d\u00e9cision n'est en attente. Reprends sans r\u00e9p\u00e9ter l'onboarding termin\u00e9
ni court-circuiter un checkpoint Design ou Implementation en attente. R\u00e9tablis
`Learning mode: active` si l'utilisateur reprend un apprentissage en pause.
Si l'onboarding est incomplet, ne pose que les questions sans r\u00e9ponse. Les
fichiers compl\u00e9mentaires manquants peuvent \u00eatre recr\u00e9\u00e9s \u00e0 partir de preuves ;
n'invente jamais l'historique d'apprentissage et n'\u00e9crase pas les notes existantes.

Si aucun profil n'existe, lis [onboarding.md](onboarding.md) et lance l'onboarding.
Utilise [state-templates.md](state-templates.md) pour cr\u00e9er l'\u00e9tat. Ces fichiers
sont du Markdown local entretenu avec les outils de fichiers habituels ; il n'y
a aucun service \u00e0 appeler.

Apr\u00e8s la configuration, continue la t\u00e2che de construction de l'utilisateur. Si
aucune n'a \u00e9t\u00e9 fournie, demande ce qu'il veut construire ou modifier. Invoquer
ce skill \u00e0 nouveau ne doit rien r\u00e9initialiser.
