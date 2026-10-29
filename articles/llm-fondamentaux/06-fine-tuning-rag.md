# Chapitre 6 — Fine-tuning et RAG

> Partie de : [LLM — fondamentaux et panorama](README.md)

**Date :** 2026-10-29

## Objectif

Savoir choisir entre **prompt engineering**, **RAG** et **fine-tuning** selon le besoin, comprendre le fonctionnement de chacun, et disposer de recettes concrètes pour mettre en œuvre le RAG et un fine-tuning léger (LoRA).

## Le décisionnaire : quelle approche pour quel besoin

Trois leviers, trois périmètres. La question n'est pas « lequel est le meilleur » mais « lequel résout mon problème au moindre coût ».

| Besoin | Solution | Pourquoi |
|--------|----------|----------|
| Mieux formuler la tâche, structurer la sortie | **Prompt engineering** (chapitre 5) | Gratuit, immédiat, réversible |
| Le modèle doit connaître **vos documents / données métier** | **RAG** | Les connaissances sont injectées à la requête, sans réentraîner |
| Le modèle doit adopter un **style, un format, un comportement** très spécifique | **Fine-tuning** (souvent LoRA) | Le comportement est appris dans les poids |
| Le modèle doit raisonner mieux sur **votre domaine** (juridique, médical, code interne) | **Fine-tuning** + RAG combinés | Les deux : connaissances (RAG) + comportement (fine-tuning) |
| Réduire le coût en tokens | Prompt engineering + RAG ciblé | Moins de contexte = moins de tokens |
| Données sensibles, pas de fuite vers un fournisseur | Modèle local + RAG local | Voir chapitre 8 et l'article [LM Studio](../lm-studio-ubuntu/README.md) |

**Règle d'or** : essayer dans l'ordre prompt → RAG → fine-tuning. Chaque étape coûte plus cher que la précédente en temps, en infrastructure et en maintenance.

## Le RAG (Retrieval-Augmented Generation)

### Principe

Le RAG consiste à **rechercher les passages pertinents** dans une base de documents, puis à les **injecter dans le prompt** du LLM, qui répond en s'ancrant sur ce contexte.

```
Question de l'utilisateur
        │
        ▼
┌─────────────────┐     ┌────────────────────────┐     ┌─────────────┐
│  1. Recherche   │────►│  2. Construction        │────►│  3. LLM     │
│  (retrieval)    │     │  du prompt              │     │  (génération│
│                 │     │                         │     │   ancrée)   │
│  « Quels morceaux│     │  Question + passages    │     │             │
│   parlent de X ?»│     │  pertinents + règles    │     │             │
└─────────────────┘     └────────────────────────┘     └─────────────┘
        ▲
        │
   Base documentaire indexée
   (vector store + éventuellement recherche hybride)
```

Le modèle ne « sait » toujours rien de plus qu'avant : on lui **donne** l'information au moment de la requête. C'est la différence fondamentale avec le fine-tuning.

### Pourquoi le RAG réduit les hallucinations

Le prompt contient une instruction du type : « Réponds UNIQUEMENT à partir des documents ci-dessous. Si l'information n'y figure pas, dis-le. » Le modèle est contraint à un corpus fermé et vérifiable (voir chapitre 3).

### Les composants d'un RAG

1. **Le découpage (chunking)** : les documents sont découpés en morceaux de ~200-800 tokens, avec un léger chevauchement (10-20 %) pour ne pas couper une idée en deux.
2. **L'embedding** : chaque morceau est transformé en vecteur dense (un modèle d'embedding, ex. : `text-embedding-3`, BGE, E5, ou un modèle local via Ollama/Sentence Transformers).
3. **L'indexation** : les vecteurs sont stockés dans une **base vectorielle** (Chroma, FAISS, Qdrant, pgvector, Pinecone, Weaviate…).
4. **La recherche** : à la requête, la question est embarquée ; on récupère les k morceaux les plus proches (similarité cosinus). k = 3 à 10 typiquement.
5. **L'augmentation du prompt** : les morceaux sont insérés dans le prompt avec la question.
6. **La génération** : le LLM répond, idéalement avec des citations des sources.

### Les variantes qui améliorent la qualité

| Technique | Principe | Quand l'utiliser |
|-----------|----------|------------------|
| **Recherche hybride** | Combiner similarité vectorielle + recherche par mots-clés (BM25) | Vocabulaire métier précis, acronymes, codes |
| **Re-ranking** | Un modèle de re-classement (cross-encoder) réordonne les candidats | Précision critique, k élevé (20-50) en entrée |
| **Requête reformulée** | Le LLM reformule la question en 2-3 variantes pour élargir la recherche | Questions ambiguës ou mal formulées |
| **RAG multi-hop** | Plusieurs tours de recherche enchaînés | La réponse dépend de plusieurs documents liés |
| **RAG avec vérification** | Un second passage vérifie que la réponse est bien fondée sur les sources | Domaine à risque (juridique, médical) |
| **Graphe de connaissances (GraphRAG)** | Indexer les relations entre entités, pas seulement les morceaux | Données fortement relationnelles |

### Exemple minimal de RAG en Python

```python
# Prérequis : pip install openai chromadb sentence-transformers
# Le vector store et l'embedding tournent en local ; seul le LLM est appelé en API.
import chromadb
from sentence_transformers import SentenceTransformer
from openai import OpenAI

embedder = SentenceTransformer("BAAI/bge-m3")          # modèle d'embedding local
client_llm = OpenAI()                                  # clé API LLM

chroma = chromadb.PersistentClient(path="./chroma_db")
collection = chroma.get_or_create_collection("docs")

def indexer(documents: list[str]) -> None:
    collection.add(
        documents=documents,
        embeddings=embedder.encode(documents).tolist(),
        ids=[f"doc-{i}" for i in range(len(documents))],
    )

def repondre(question: str, k: int = 5) -> str:
    resultats = collection.query(
        query_embeddings=embedder.encode([question]).tolist(),
        n_results=k,
    )
    contexte = "\n\n---\n\n".join(resultats["documents"][0])

    reponse = client_llm.chat.completions.create(
        model="gpt-5-mini",
        temperature=0,
        messages=[
            {"role": "system", "content": (
                "Réponds UNIQUEMENT à partir des documents fournis. "
                "Si la réponse n'y figure pas, réponds « Je ne sais pas ». "
                "Cite tes sources entre crochets [doc n]."
            )},
            {"role": "user", "content": f"Documents :\n{contexte}\n\nQuestion : {question}"},
        ],
    )
    return reponse.choices[0].message.content
```

### Pièges du RAG

- **« Garbage in, garbage out » sur les documents** : un RAG sur des documents obsolètes ou mal structurés donne des réponses fausses *avec l'air de sources fiables*.
- **Le mauvais morceau** : si la recherche récupère le mauvais passage, le modèle répond avec assurance sur le mauvais sujet.
- **Le contexte trop long** : 10 morceaux x 800 tokens = 8 000 tokens de contexte à chaque requête. Coût et latence montent vite. D'où l'intérêt du re-ranking et d'un k raisonnable.
- **Les tableaux, images et mises en page** : le découpage naïf détruit les tableaux et les schémas. Utiliser des parseurs adaptés (PDF → markdown structuré, tableaux en CSV/JSON).

## Le fine-tuning

### Principe

Le **fine-tuning** continue l'entraînement du modèle (chapitre 2, phases 2-4) **sur vos propres données**, pour modifier son comportement de façon persistante.

- **Fine-tuning complet** : tous les poids sont ajustés. Puissant, mais coûteux (il faut charger le modèle entier en mémoire GPU) et risqué (oubli catastrophique : le modèle oublie ce qu'il savait).
- **PEFT / LoRA (Low-Rank Adaptation)** : on fige le modèle et on n'entraîne qu'une petite matrice de rang faible ajoutée à chaque couche. **~1 % des paramètres** ajustés, résultats souvent équivalents pour l'adaptation de style/format. C'est la méthode standard en 2026.

### Quand fine-tuner plutôt que faire du RAG

| Fine-tuning | RAG |
|-------------|-----|
| Apprendre un **comportement** (style, ton, format, réflexes) | Fournir des **connaissances** (faits, documents) |
| Le savoir-faire doit être **persistant** dans le modèle | Les connaissances **changent souvent** (il suffit de réindexer) |
| Coût fixe à l'entraînement, coût d'inférence inchangé | Coût au token (contexte plus long), maintenance de l'index |
| Nouvelle version du modèle à déployer et à maintenir | Mise à jour des documents sans réentraînement |

**La plupart des projets métiers ont besoin des deux** : RAG pour les connaissances à jour, LoRA pour le style et les réflexes métier.

### Recette LoRA avec les outils standards

```bash
# Prérequis : pip install transformers peft datasets accelerate trl torch
# Nécessite un GPU avec suffisamment de VRAM (ou du QLoRA, voir ci-dessous).
```

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import LoraConfig
from trl import SFTTrainer
from datasets import load_dataset

model_id = "Qwen/Qwen3-8B"  # modèle de base open weights

tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id, device_map="auto")

lora = LoraConfig(
    r=16,                # rang de la décomposition (8-64 typiquement)
    lora_alpha=32,       # échelle (souvent 2 x r)
    target_modules=["q_proj", "v_proj"],  # couches à adapter
    lora_dropout=0.05,
    task_type="CAUSAL_LM",
)

# dataset au format "conversations" : une colonne avec l'échange complet
dataset = load_dataset("json", data_files="mes_exemples.jsonl")["train"]

trainer = SFTTrainer(
    model=model,
    processing_class=tokenizer,
    train_dataset=dataset,
    args={"output_dir": "./lora-out", "num_train_epochs": 3,
          "per_device_train_batch_size": 4, "learning_rate": 2e-4},
    peft_config=lora,
)
trainer.train()
trainer.save_model("./lora-out")
```

**QLoRA** : version quantifiée du LoRA — le modèle de base est chargé en 4 bits, ce qui permet de fine-tuner un modèle de 8-14B sur un GPU grand public (ex. : RTX 4090, 24 Go), voire de 30-70B sur du matériel plus conséquent.

### Préparer les données de fine-tuning

La qualité des données prime sur la quantité (chapitre 2) :

- **500 à 5 000 exemples** bien construits suffisent pour un LoRA efficace sur un style/format.
- Format : paires (instruction, réponse idéale), couvrant les **cas normaux ET les cas limites** (erreurs fréquentes, refus attendus, format spécial).
- **Dédupliquer**, vérifier l'orthographe, s'assurer que les réponses sont réellement ce que vous voulez.
- Mélanger avec ~10-20 % d'exemples génériques pour ne pas faire oublier au modèle ses capacités de base.

### Héberger et servir un modèle fine-tuné

- **Via API fournisseur** : la plupart (OpenAI, Mistral, Together, Fireworks…) permettent d'uploader un fine-tune. Le plus simple.
- **En local** : merger les poids LoRA avec le modèle de base, puis servir avec vLLM, Ollama ou llama.cpp (voir [LM Studio](../lm-studio-ubuntu/README.md) et [colibri](../colibri/README.md)).

## Combiner RAG + fine-tuning + prompt engineering

L'architecture cible pour un projet métier sérieux :

```
System prompt (rôle, règles, garde-fous)      ← prompt engineering
        │
        ▼
Contexte RAG (documents métier à jour)        ← RAG
        │
        ▼
Modèle fine-tuné (style, réflexes, format)    ← LoRA
        │
        ▼
Post-traitement (validation, citations)       ← code déterministe
```

Chaque couche a un rôle distinct et réversible. C'est plus maintenable qu'un seul gros système.

## Pièges connus

- **Fine-tuner pour des faits** : le modèle oublie, hallucine toujours, et les faits deviennent obsolètes. Le RAG est fait pour ça.
- **RAG pour du style** : injecter 10 exemples de style dans le prompt coûte cher et marche moyennement. Le LoRA est fait pour ça.
- **Sous-estimer la maintenance du RAG** : réindexer à chaque mise à jour documentaire, surveiller la qualité de recherche, versionner les embeddings.
- **Le fine-tuning qui « dégrade » le modèle** : trop de données bruitées ou un learning rate trop haut = modèle moins bon partout. Toujours évaluer avant/après sur un jeu de test.
- **Négliger l'évaluation** : un fine-tune sans mesure de qualité est un pari. Benchmarker sur 50-200 cas avant de déployer (même méthode que chapitre 5).

## Références

- [Hugging Face — « Fine-tuning » (Transformers)](https://huggingface.co/docs/transformers/training) — guide officiel
- [Hugging Face — PEFT (LoRA/QLoRA)](https://huggingface.co/docs/peft/index) — documentation de la bibliothèque de référence
- [Hugging Face — TRL (SFTTrainer)](https://huggingface.co/docs/trl/index) — entraînement supervisé simplifié
- [LangChain — « RAG »](https://python.langchain.com/docs/tutorials/rag/) — tutoriel RAG pas à pas
- [LlamaIndex — « RAG »](https://docs.llamaindex.ai/en/stable/) — framework RAG (chunking, retrieval, évaluation)
- [Pinecone — « What is RAG? »](https://www.pinecone.io/learn/retrieval-augmented-generation/) — introduction pédagogique (en anglais)
- [Chroma — « RAG guide »](https://www.trychroma.com/blog) — base vectorielle légère, bonne pour démarrer
