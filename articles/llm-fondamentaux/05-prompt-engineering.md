# Chapitre 5 — Le prompt engineering

> Partie de : [LLM — fondamentaux et panorama](README.md)

**Date :** 2026-10-29

## Objectif

Savoir rédiger un prompt structuré et efficace — rôle, contexte, instruction, format de sortie — appliquer les techniques éprouvées (exemples, chaîne de pensée, balises) et mettre en place une démarche d'itération mesurée.

## Définition

Le **prompt engineering** est l'art de formuler les instructions envoyées à un LLM pour obtenir le résultat voulu, **sans modifier les poids du modèle**. C'est le levier le plus rapide, le moins cher et le plus réversible : on change du texte, on reteste, on garde ce qui marche.

Ordre de priorité des leviers (du plus simple au plus lourd) :

```
1. Prompt engineering   →  heures, gratuit, réversible
2. RAG (chapitre 6)     →  jours, coût infra, réversible
3. Fine-tuning (chapitre 6) → jours/semaines, coût GPU, nouvelle version du modèle
```

**Toujours commencer par le prompt engineering.** Passer au RAG ou au fine-tuning avant d'avoir un prompt solide gaspille du temps et de l'argent.

## Anatomie d'un bon prompt

Un prompt efficace suit une structure en quatre blocs. Même les modèles récents, très bons pour comprendre le langage naturel, bénéficient énormément d'une structure explicite.

```
┌─────────────────────────────────────────────┐
│  RÔLE (system prompt)                       │  Qui est le modèle, dans quel ton
│  « Tu es un réviseur de code senior… »       │
├─────────────────────────────────────────────┤
│  CONTEXTE                                   │  Pourquoi la tâche existe, pour qui,
│  « L'utilisateur est développeur junior… »   │  quelles sont les contraintes
├─────────────────────────────────────────────┤
│  INSTRUCTION (tâche)                        │  Verbe d'action + périmètre précis
│  « Relis ce code et signale les bugs… »      │
├─────────────────────────────────────────────┤
│  FORMAT DE SORTIE                           │  Structure exacte attendue
│  « Réponds en JSON : {bugs: [...], …} »     │
└─────────────────────────────────────────────┘
        +  EXEMPLES (few-shot) en annexe si besoin
        +  DONNÉES (le contenu à traiter)
```

### Exemple complet

```markdown
Tu es un assistant spécialisé dans la rédaction de notes de frais
professionnelles, pour des commerciaux en déplacement.

RÈGLES :
- Ton professionnel mais concis (2-3 phrases maximum par champ).
- Si une information manque, écris « À COMPLÉTER » plutôt que d'inventer.
- Ne jamais inventer de montants, de dates ou de noms de clients.

TÂCHE :
À partir de la note de frais brute ci-dessous, produis un récapitulatif
structuré.

FORMAT DE SORTIE (JSON strict, rien d'autre) :
{
  "destination": "ville",
  "date_debut": "AAAA-MM-JJ",
  "date_fin": "AAAA-MM-JJ",
  "motif": "1 phrase",
  "depenses": [{"type": "transport|hotel|restauration", "montant_estime": "à défaut, null"}],
  "points_a_clarifier": ["..."]
}

NOTE DE FRAIS BRUTE :
<note>
Départ mardi pour Lyon, retour jeudi. Train AR 120 €, hôtel 2 nuits
x 95 €, 3 restos avec clients. Le client Durand a annulé mercredi,
j'ai quand même pris l'hôtel.
</note>
```

## Les règles d'or

1. **Être spécifique.** « Sois clair » ne veut rien dire. « Écris des phrases de moins de 20 mots, vocabulaire de niveau lycée » est actionnable.
2. **Montrer le format de sortie.** « Réponds en JSON » est faible ; un exemple JSON concret ou un schéma est fort. Quand l'API le permet, utiliser `response_format` (chapitre 4) pour le garantir.
3. **Délimiter les responsabilités.** « Ne réponds que sur le document fourni », « Si tu ne sais pas, dis-le » (voir chapitre 3).
4. **Un test humain simple** : montrez votre prompt à un collègue qui ne connaît pas le sujet. S'il ne peut pas exécuter la tâche sans vous poser de questions, le modèle non plus.
5. **Le contexte bat la longueur.** Un prompt de 150 mots bien ciblés bat un pavé de 2 000 mots confus. Chaque token coûte de l'argent et de la latence.

## Les techniques principales

### Le rôle (system prompt)

Définir un rôle concentre le comportement du modèle sur un domaine.

```
Tu es un ingénieur senior en revue de code, spécialisé Python.
Tu privilégies la lisibilité et la sûreté sur la concision.
```

Utile pour le ton et les priorités. À ne pas surcharger : un rôle ne remplace pas une instruction précise.

### Le few-shot (exemples dans le prompt)

Montrer 1 à 5 exemples du format attendu. C'est souvent la technique la plus efficace pour le formatage.

```
Classe le sentiment de chaque avis (positif / négatif / neutre).

Avis : « Livraison rapide, produit conforme. » → positif
Avis : « Colis arrivé ouvert, service client injoignable. » → négatif
Avis : « Reçu dans les délais. » → neutre
Avis : « {{AVIS}} » →
```

- **0 exemple** = zero-shot, **1-5** = few-shot. Au-delà, les gains diminuent et le coût en tokens grimpe.
- Choisir des exemples **représentatifs des cas difficiles**, pas seulement les faciles.

### La chaîne de pensée (Chain-of-Thought)

Demander au modèle de raisonner étape par étape avant de conclure. Sur les tâches de raisonnement (maths, logique, analyse), le gain est significatif.

```
Analyse ce contrat étape par étape :
1. Identifie les clauses inhabituelles.
2. Pour chacune, explique le risque.
3. Conclus par une recommandation (accepter / négocier / refuser).
```

**Variante structurée** : demander d'abord l'analyse, puis la conclusion dans un format séparé. Les modèles récents de raisonnement (« extended thinking ») intègrent ce mécanisme nativement — activer leur mode réflexion quand la tâche est complexe.

### Les délimiteurs et balises

Quand le prompt mélange plusieurs types de contenu (instructions, contexte, données), les séparer visuellement réduit les confusions :

```markdown
<instructions>
Résume le document ci-dessous en 3 points.
</instructions>

<document>
{{CONTENU_DU_DOCUMENT}}
</document>

Règles : ne cite que le document. Si un point n'y figure pas, omets-le.
```

Conventions utiles : balises XML simples (`<document>`, `<exemple>`), ou séparateurs (`###`, `---`). L'important est la **cohérence** : les mêmes balises partout, dans tous les prompts de l'application.

### Décomposer les tâches (prompt chaining)

Une tâche complexe en une seule étape donne un résultat moyen. La découper en étapes, chacune avec son propre prompt, est plus fiable :

```
Étape 1 : extraire les entités (dates, montants, noms) du texte brut
Étape 2 : valider les entités extraites contre des règles métier
Étape 3 : rédiger le résumé à partir des entités validées
```

Chaque étape est plus simple, donc plus fiable, et chaque sortie intermédiaire est vérifiable.

### Formuler en positif

Dire ce qu'il faut faire plutôt que ce qu'il ne faut pas faire :

- ❌ « Ne sois pas verbeux. » → le modèle entend « verbeux ».
- ✅ « Réponds en 2 phrases maximum. »

## Le prompt système vs le prompt utilisateur

| | System prompt | Message utilisateur |
|---|---|---|
| Rôle | Définit le comportement général, le rôle, les règles permanentes | Porte la tâche concrète et les données |
| Persistance | Envoyé à **chaque** appel de la conversation | Change à chaque tour |
| Coût | Payé à chaque requête (attention aux prompts système trop longs) | Payé une fois par message |
| Bonne pratique | Rôle + règles + format global + garde-fous | Tâche + données + contexte spécifique au tour |

**Piège** : un system prompt trop long et trop chargé « noie » les instructions. Garder le system prompt focalisé ; mettre les détails de la tâche dans le message utilisateur.

## Itérer et évaluer : la méthode

Le prompt engineering est un **processus expérimental**, pas une inspiration.

1. **Partir d'une version simple** (rôle + tâche + format).
2. **Créer un jeu de test** : 20-50 entrées représentatives, avec les sorties attendues (même partielles).
3. **Tester, noter les échecs** : catégoriser (format faux, contenu faux, ton faux, hallucination…).
4. **Modifier UN élément à la fois** : changer deux choses et vous ne saurez pas laquelle a marché.
5. **Re-tester sur le même jeu.** Revenir en arrière si ça régresse.
6. **Versionner les prompts** : les stocker dans des fichiers (YAML/Markdown), pas dans le code en dur. Un prompt est un artefact de production.

### Grille d'évaluation rapide (sans vérité terrain complète)

| Critère | Comment évaluer |
|---------|-----------------|
| Fidélité à la tâche | La sortie fait-elle ce qui était demandé ? |
| Format | Structure respectée ? (testable automatiquement pour du JSON) |
| Factualité | Les affirmations sont-elles sourcées / vérifiables ? |
| Hallucinations | Le modèle a-t-il inventé ? (voir chapitre 3) |
| Ton / style | Conforme au rôle demandé ? |
| Coût | Nombre de tokens entrée + sortie par requête |

### Exemple de fiche de test

```yaml
# tests/resume-contrat.yaml
- id: resume-01
  entree: "<texte de contrat court>"
  attendu:
    nb_points_max: 3
    contient_date_signature: true
  a_verifier_humain: true
```

## Les pièges classiques

- **Le « prompt magique » copié-collé** : un prompt trouvé sur Internet est optimisé pour un modèle et un tâche précis. Toujours l'adapter et le re-tester.
- **Sur-instruire** : 30 règles contradictoires produisent un modèle perdu. Prioriser ; déplacer les règles secondaires dans des exemples plutôt que dans des interdictions.
- **Négliger les cas limites** : un prompt qui marche sur les exemples « propres » échoue en production sur les entrées bizarres (texte vide, langue étrangère, injection). Tester les cas limites explicitement.
- **L'injection de prompt** : un utilisateur malveillant peut inclure dans ses données des instructions du type « ignore les instructions précédentes ». Défenses : séparer clairement instructions et données (balises), ne jamais faire confiance au contenu utilisateur comme s'il était une instruction (voir chapitre 7 sur les agents).
- **Oublier de re-tester après un changement de modèle** : un prompt optimisé pour un modèle n'est pas optimal pour un autre.

## Quand le prompt engineering ne suffit plus

Signaux qu'il faut passer à la vitesse supérieure (chapitre 6) :

- Le modèle ne parvient pas à adopter un **style ou un format très spécifique**, même avec beaucoup d'exemples.
- Il lui manque des **connaissances métier pointues** que le contexte ne peut pas fournir à chaque requête.
- Le taux d'hallucination reste élevé malgré RAG et température basse.
- Le coût du few-shot (dizaines d'exemples à chaque requête) devient prohibitif.

## Références

- [Anthropic — « Prompt engineering overview »](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — guide officiel, techniques à jour (en anglais)
- [Anthropic — « Prompt engineering best practices »](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/claude-prompting-best-practices) — balises, rôle, exemples (en anglais)
- [Prompting Guide — promptingguide.ai](https://www.promptingguide.ai/) — ressource communautaire complète (en anglais)
- [OpenAI — « Prompt engineering »](https://platform.openai.com/docs/guides/prompt-engineering) — stratégies officielles (en anglais)
- [Anthropic — tutoriel interactif de prompt engineering (GitHub)](https://github.com/anthropics/prompt-eng-interactive-tutorial) — exercices guidés (en anglais)
- [Lilian Weng — « Prompt Engineering »](https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/) — synthèse académique (en anglais)
