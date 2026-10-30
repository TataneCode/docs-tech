# Chapitre 9 — Comment fonctionne un réseau de neurones

> Partie de : [LLM — fondamentaux et panorama](README.md)

**Date :** 2026-10-29

## Objectif

Expliquer le fonctionnement interne d'un réseau de neurones artificiel — neurone, couches, propagation avant, rétropropagation — pour comprendre ce que signifient « poids », « paramètres » et « entraînement » quand on parle d'un LLM (chapitre 1) et de son cycle de vie (chapitre 2).

## Du neurone biologique au neurone artificiel

L'analogie de départ : un neurone biologique reçoit des signaux via ses dendrites, les intègre, et émet un signal via son axone si le total dépasse un seuil. Le **neurone artificiel** (ou perceptron) reprend cette idée sous forme de calcul :

```
x1 ──w1──┐
x2 ──w2──┼──>  Σ (+ b)  ──>  f(·)  ──>  sortie a
x3 ──w3──┘

z = w1·x1 + w2·x2 + w3·x3 + b     (somme pondérée + biais)
a = f(z)                           (fonction d'activation)
```

- **Les entrées** `x1, x2, x3` : les données (pixels, caractéristiques, vecteur d'un token…).
- **Les poids** `w1, w2, w3` et le **biais** `b` : les paramètres du neurone, ajustés pendant l'entraînement.
- **La fonction d'activation** `f` : introduit une non-linéarité. Sans elle, empiler des neurones resterait une simple transformation linéaire, incapable de représenter des frontières complexes.

Fonctions d'activation courantes :

| Fonction | Formule | Usage |
|----------|---------|-------|
| Sigmoïde | `1 / (1 + e^-z)` | Historique ; aujourd'hui évitée dans les couches cachées (gradient qui s'évanouit) |
| Tanh | `tanh(z)` | Variante centrée de la sigmoïde |
| ReLU | `max(0, z)` | La plus répandue : simple, rapide, robuste au gradient évanescent |
| GELU / SwiGLU | variantes lisses | Standard dans les transformers modernes (meilleure qualité, coût un peu plus élevé) |

## Des neurones aux couches : le perceptron multicouche

Un **réseau de neurones** assemble des neurones en couches connectées : une couche d'entrée, une ou plusieurs couches cachées, une couche de sortie. Dans une couche « dense » (fully connected), chaque neurone est connecté à tous les neurones de la couche suivante.

- **Profondeur** = nombre de couches ; **largeur** = nombre de neurones par couche.
- Chaque connexion porte un poids : c'est l'ensemble de ces poids qui constitue les **paramètres** du réseau. « Un modèle de 7 milliards de paramètres » signifie 7 milliards de poids (et biais), pas 7 milliards de neurones.
- Une couche entière se calcule en une opération : `y = f(W·x + b)`, où `W` est la matrice des poids. D'où l'importance des GPU : ce sont des multiplications de matrices massivement parallélisables.

## Propagation avant : calculer une sortie

La **propagation avant** (forward pass) enchaîne les couches de gauche à droite. Exemple : un réseau qui estime une note sur 10 à partir de trois critères normalisés (scénario, jeu des acteurs, effets spéciaux) :

```
Entrée        : x = [0.9, 0.4, 0.8]
Couche cachée : z = W1·x + b1   puis   a = ReLU(z)
Sortie        : y = W2·a + b2   →  note prédite
```

À l'initialisation, les poids sont tirés aléatoirement : la sortie est du bruit. C'est l'entraînement qui transforme ce bruit en prédictions utiles.

## La fonction de perte : mesurer l'erreur

La **fonction de perte** (loss) quantifie l'écart entre la sortie prédite et la sortie attendue :

- **Erreur quadratique moyenne** pour la régression (prédire un nombre).
- **Entropie croisée** pour la classification — et c'est elle qu'utilisent les LLM : la sortie est une distribution de probabilités sur tout le vocabulaire, et la cible est le vrai token suivant (chapitre 1).

## Rétropropagation : apprendre de ses erreurs

L'entraînement consiste à ajuster chaque poids pour réduire la perte. La **rétropropagation** calcule, pour chaque poids, de combien il contribue à l'erreur — le gradient `∂L/∂w` — en appliquant la règle de la chaîne de la sortie vers l'entrée, puis met à jour les poids dans le sens opposé :

```
w ← w − η · ∂L/∂w        (η = taux d'apprentissage, learning rate)
```

La boucle d'entraînement, répétée sur des millions de lots (batches) d'exemples :

```
pour chaque lot d'exemples :
    1. propagation avant   : calculer la prédiction
    2. calculer la perte
    3. rétropropagation     : calculer les gradients de tous les poids
    4. mise à jour          : w ← w − η · gradient
```

Le **taux d'apprentissage** `η` est le réglage le plus sensible : trop grand, l'apprentissage diverge ou oscille ; trop petit, il est extrêmement lent. En pratique, on utilise un optimiseur comme **Adam** (taux adaptatif par paramètre, avec momentum) et un planning de `η` (warmup puis décroissance).

## Ce que « apprendre » signifie concrètement

Le réseau ne mémorise pas les exemples un par un : il ajuste ses milliards de poids pour que la fonction globale approxime au mieux la distribution des données. C'est ce qui lui permet de **généraliser** à des exemples jamais vus — et aussi d'apprendre les biais et les erreurs du corpus (voir chapitres 2 et 3).

Analogie : apprendre à reconnaître un chat, c'est ajuster les poids pour que « photo avec oreilles pointues et moustaches » produise une forte probabilité « chat », sans avoir stocké aucune photo.

## Du perceptron au transformer

Le transformer des LLM (chapitre 1) réutilise exactement ces briques :

- **Embedding** : chaque token est converti en un vecteur dense (ex. 4 096 dimensions) via une matrice de poids apprise.
- **Bloc transformer** = deux sous-briques empilées :
  - **l'attention** (chapitre 1) : mélange l'information entre les tokens ;
  - **le MLP** (perceptron multicouche à 2 couches avec activation) : appliqué à chaque token indépendamment.
- Un LLM = N blocs identiques (typiquement 32 à 128), plus la tête de sortie qui projette sur le vocabulaire.

Les poids d'un LLM sont donc : la matrice d'embedding + les poids de chaque bloc (matrices Q, K, V, O de l'attention + poids du MLP) + la tête de sortie. Ordres de grandeur : un modèle « 7B » = 7 milliards de poids ≈ 14 Go en FP16 ; un « 70B » ≈ 140 Go en FP16 — d'où la quantification (chapitre 10).

## L'entraînement à grande échelle (aperçu)

Passer d'un réseau jouet à un LLM demande une infrastructure considérable (détails et coûts au chapitre 2) :

- **Données** : des centaines de milliards à des billions de tokens, nettoyés et dédupliqués.
- **Matériel** : des milliers de GPU (NVIDIA H100/B200…) interconnectés (NVLink, InfiniBand) dans un data center.
- **Parallélisme** : les données sont réparties sur les GPU (data parallel), et le modèle lui-même est découpé (tensor parallel, pipeline parallel, ZeRO pour la mémoire de l'optimiseur).
- **Scaling laws** : la perte diminue selon des lois de puissance prévisibles en fonction des données, des paramètres et du calcul — ce qui guide les décisions d'investissement des laboratoires.

C'est pourquoi l'entraînement complet est réservé aux laboratoires (chapitre 2, phase 1), et pourquoi les utilisateurs se contentent d'inférence (chapitre 10) ou de fine-tuning léger (chapitre 6).

## Pièges connus

- **« C'est de la magie » / « c'est juste des maths »** : ni l'un ni l'autre. Le réseau calcule une fonction déterministe, très complexe ; ce qui surprend, c'est l'émergence de comportements jamais explicitement programmés.
- **Confondre paramètres et neurones** : « 7 milliards de neurones » est un abus de langage pour 7 milliards de poids.
- **Croire que le modèle « stocke » ses données d'entraînement** : il compresse des régularités statistiques dans ses poids. Une mémorisation partielle de fragments existe néanmoins (risque de fuite de données, voir chapitre 3).
- **Sous-estimer le rôle des données** : aujourd'hui, plus de données de qualité bat souvent plus de paramètres ; un « petit » modèle bien entraîné dépasse un « gros » modèle mal entraîné.

## Références

- [3Blue1Brown — « But what is a Neural Network? »](https://www.3blue1brown.com/lessons/neural-networks/) — introduction visuelle, accessible sans maths (en anglais)
- [3Blue1Brown — série Deep Learning (réseau, descente de gradient, rétropropagation)](https://www.3blue1brown.com/topics/neural-networks) — la meilleure visualisation de l'entraînement (en anglais)
- [Michael Nielsen — « Neural Networks and Deep Learning » (livre en ligne gratuit)](http://neuralnetworksanddeeplearning.com/) — chapitre 1 (perceptron) et chapitre 3 (rétropropagation), avec code Python exécutable
- [Andrej Karpathy — « Neural Networks: Zero to Hero »](https://github.com/karpathy/nn-zero-to-hero) — reconstruire un mini-GPT from scratch en Python (en anglais)
- [« Attention Is All You Need » — papier fondateur du transformer (arXiv)](https://arxiv.org/abs/1706.03762)
- [LoRA — Low-Rank Adaptation, fine-tuning efficace (arXiv)](https://arxiv.org/abs/2106.09685) — voir chapitre 6
- [Chapitre 1 de cette série](01-fonctionnement.md) — le transformer et l'attention au-dessus de ces briques
