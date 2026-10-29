# Chapitre 3 — Les hallucinations

> Partie de : [LLM — fondamentaux et panorama](README.md)

**Date :** 2026-10-29

## Objectif

Définir précisément ce qu'est une hallucination, comprendre pourquoi les LLM en produisent, et appliquer les techniques qui réduisent le risque dans une application réelle.

## Définition

Une **hallucination** est une sortie d'un LLM qui :

- est **fluente et convaincante**, mais
- est **factuellement fausse, inventée ou non fondée** sur le contexte fourni.

Exemples typiques :

- Citer un article scientifique, un arrêt de jurisprudence ou un livre **qui n'existe pas**.
- Donner une date, un chiffre ou une statistique **inventée**.
- Affirmer avec assurance quelque chose de **faux** (ex. : « Paris est la capitale de l'Espagne »).
- Résumer un document en y ajoutant des **détails absents** du document.

**Point clé** : le modèle ne « sait » pas qu'il hallucine. Il génère le texte le plus *plausible*, pas le plus *vrai*. La fluidité n'est pas un indicateur de fiabilité.

## Pourquoi les LLM hallucinent

Trois causes structurelles (voir chapitres 1 et 2) :

1. **L'objectif d'entraînement est la plausibilité, pas la vérité.** Le modèle apprend « quel token suit le plus probablement », pas « quel fait est vérifié ». Un nom d'auteur inventé mais crédible a une probabilité élevée.
2. **La connaissance est compressée et imparfaite.** Les faits sont stockés sous forme de patterns statistiques, pas de base de connaissances. Les détails rares sont les premiers à être « oubliés » ou reconstruits de travers.
3. **La date de coupure.** Le modèle ne connaît rien de ce qui est arrivé après son corpus d'entraînement. Toute question sur l'actualité déclenche un risque d'invention.

Un quatrième facteur, plus subtil : **l'alignement pousse à répondre**. Un modèle aligné est entraîné à être utile et à ne pas refuser. Face à une question à laquelle il ne « sait » pas répondre, il préfère souvent inventer une réponse plausible plutôt que de dire « je ne sais pas » — surtout si le prompt ne l'y autorise pas explicitement.

## Les différents types d'hallucinations

| Type | Description | Exemple |
|------|-------------|---------|
| **Intrinsèque** | Contradit le contexte fourni dans le prompt | Le document dit « contrat signé en 2023 », le résumé indique « 2024 » |
| **Extrinsèque** | Inventé de toutes pièces, invérifiable dans le contexte | Une fausse référence bibliographique |
| **Fausses attributions** | Action ou parole attribuée à la mauvaise personne | « Selon Einstein, … » pour une citation apocryphe |
| **Chiffres inventés** | Statistiques, montants, pourcentages sans source | « 73 % des entreprises » sans étude derrière |
| **Confabulation conversationnelle** | Le modèle « oublie » ce qu'il a dit plus tôt et se contredit | Deux réponses successives incompatibles dans un même chat |

## Comment réduire les hallucinations

### 1. Autoriser explicitement l'abstention

La mesure la plus efficace et la moins coûteuse. Dans le prompt système :

```
Si tu n'es pas sûr d'un fait, ou si l'information n'est pas dans le
contexte fourni, réponds « Je ne sais pas » plutôt que d'inventer.
Ne cite jamais une source que tu ne peux pas vérifier.
```

### 2. Ancrer sur un contexte fourni (RAG)

Fournir les documents pertinents dans le prompt et interdire de sortir de ce cadre :

```
Réponds UNIQUEMENT à partir des documents ci-dessous. Si la réponse
n'y figure pas, dis-le.

<documents>
{{DOCUMENTS}}
</documents>

Question : {{QUESTION}}
```

C'est le principe du **RAG** (Retrieval-Augmented Generation, voir chapitre 6). Le taux d'hallucination chute drastiquement quand le modèle est contraint à un corpus fermé.

### 3. Demander des citations vérifiables

```
Pour chaque affirmation factuelle, cite le passage exact du document
source entre guillemets, avec le nom du document.
```

Puis **vérifier mécaniquement** : le passage cité existe-t-il vraiment dans le document ? (Un simple test de chaîne de caractères élimine beaucoup d'hallucinations de citation.)

### 4. Baisser la température

Une température basse (0 à 0,3) réduit la créativité — donc l'invention (voir chapitre 4). Pour toute tâche factuelle, rester bas.

### 5. Décomposer la tâche

Demander au modèle de **raisonner étape par étape** (« réfléchis avant de répondre ») améliore la fiabilité sur les tâches complexes. Mieux : séparer **recherche** et **rédaction** en deux appels distincts.

### 6. La vérification externe (grounding)

- **Recherche web** : connecter le modèle à un moteur de recherche pour qu'il ancre ses réponses (tool use, chapitre 7).
- **Vérification croisée** : faire relire la réponse par un second modèle (ou un second passage) en lui demandant de signaler les affirmations non sourcées.
- **Déterminisme sur les faits** : pour les données structurées (dates, montants, codes), ne jamais laisser le LLM les générer librement — les injecter depuis une base de données.

### 7. Le fine-tuning ciblé

Fine-tuner sur des exemples du domaine réduit les hallucinations **dans ce domaine** (chapitre 6). À réserver aux cas où le prompting + RAG ne suffisent pas.

## Évaluer le taux d'hallucination

On ne peut pas « faire confiance » sans mesurer. Méthodes concrètes :

- **Jeu de test golden set** : un ensemble de 50-200 questions dont vous connaissez la bonne réponse (ou l'absence de réponse). Mesurer le taux de réponses correctes, fausses, et « je ne sais pas ».
- **Vérification humaine** : échantillonner les sorties et les faire noter (coûteux, réservé à la validation).
- **Vérification automatique** : pour les citations, vérifier que les passages existent ; pour les entités (noms, dates), les comparer à une base de référence.

**Métrique utile** : taux d'hallucination = nombre d'affirmations factuelles fausses / nombre total d'affirmations factuelles. Un bon système RAG descend sous les 2-5 % sur son domaine ; un LLM brut peut dépasser 20-30 % sur les questions factuelles pointues.

## Hallucination ou pas ? Cas limites

- **Résumé fidèle mais incomplet** : ce n'est pas une hallucination, c'est un biais de sélection. Le résumeur choisit quoi garder.
- **Interprétation raisonnable d'un texte ambigu** : le LLM « prend position » là où un humain hésiterait. Formuler la question plus précisément.
- **Réponse obsolète** : le fait était vrai à la date d'entraînement, faux aujourd'hui. Ce n'est pas une hallucination au sens strict, mais le risque est identique — d'où l'intérêt de la recherche web en temps réel.

## Pièges connus


- **« Le modèle a cité une source, donc c'est vrai »** : les citations sont le domaine d'hallucination le plus fréquent. Toujours vérifier.
- **La confiance affichée trompe** : un modèle hésitant et un modèle halluciné peuvent paraître aussi assurés l'un que l'autre. Ne jamais utiliser le ton comme signal de fiabilité.
- **Les petits modèles hallucinent plus** : à tâche égale, un modèle de 3B inventera plus qu'un frontier. Ne pas lésiner sur la taille pour les tâches factuelles critiques.
- **Le RAG n'élimine pas le risque** : il le réduit fortement, mais le modèle peut encore mal interpréter les documents ou « compléter » au-delà.

## Références

- [Hugging Face — « LLM Hallucinations » (blog)](https://huggingface.co/blog/llm-hallucinations) — panorama des causes et solutions (en anglais)
- [Vectara — « Hallucination Leaderboard »](https://github.com/vectara/hallucination-leaderboard) — classement public des modèles sur la fidélité au résumé (en anglais)
- [Anthropic — « Claude's system prompt » (exemples)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — techniques d'ancrage (en anglais)
- [Stanford CRFM — HELM](https://crfm.stanford.edu/helm/latest/) — benchmarks incluant la fiabilité factuelle (en anglais)
- [Article académique — « A Survey on Hallucination in LLMs » (arXiv)](https://arxiv.org/abs/2311.05232) — revue complète (en anglais)
