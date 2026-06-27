# Week 12: Theory - Vector Embeddings and Semantic Enrichment

Through week 11 you encoded data with **structured features**: one-hot, ordinal, target-encoded, hashed. They work - but they hit a ceiling on **text columns**, especially long ones (product descriptions, support tickets, contract clauses). rapidfuzz (week 08) catches typos and reorderings; it doesn't catch "Apple AirPods Pro" matching "wireless earphones from Apple."

**Embeddings** are the modern answer: convert text (or image, or audio) into a dense vector that captures semantic similarity. Two items with related meaning land near each other in the vector space, regardless of surface differences.

This week is the data-engineering side of embeddings: how to generate them, store them as columns, build a vector index, and use them for **semantic dedup**, **semantic search**, and **semantic classification**. Plus the modern decision: **embed-then-classical-ML** vs **prompt-the-LLM**.

---

## Part 1: What an embedding is

An **embedding model** is a function from text (or image) to a fixed-length vector:

```
"Apple AirPods Pro"      → [0.12, -0.41, ..., 0.08]   (768 dimensions)
"Apple wireless earbuds" → [0.13, -0.39, ..., 0.09]   (very similar!)
"raincoat"               → [-0.21, 0.55, ..., -0.03]  (totally different)
```

The model is trained so that **cosine similarity** between vectors reflects semantic similarity. Cosine ranges from -1 (opposite) to 1 (identical); for text-trained models, related items are usually >0.6, unrelated <0.3.

### Dimensions and quality

| Model | Dimensions | Quality |
|---|---|---|
| `all-MiniLM-L6-v2` | 384 | Fast; good for English; baseline |
| `all-mpnet-base-v2` | 768 | Better quality; slower |
| `bge-large-en-v1.5` | 1024 | High quality; SOTA on MTEB |
| OpenAI `text-embedding-3-small` | 1536 | Commercial; very good |
| OpenAI `text-embedding-3-large` | 3072 | Commercial; SOTA |
| Voyage `voyage-3` | 1024 | Commercial; competitive |

Higher dimensions = better fidelity but bigger storage. For 1M rows × 1024 dim × 4 bytes = **4 GB of embeddings**. Plan accordingly.

### Where to look for the right model

The **MTEB leaderboard** ([huggingface.co/spaces/mteb/leaderboard](https://huggingface.co/spaces/mteb/leaderboard)) ranks embedding models on retrieval, classification, clustering. **Sort by your task; pick the smallest model in the top 5.** Bigger isn't always better - overshooting costs latency and memory.

---

## Part 2: sentence-transformers - the open standard

The library most people use:

```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("all-MiniLM-L6-v2")

texts = ["Apple AirPods Pro", "Apple wireless earbuds", "raincoat"]
embeddings = model.encode(texts)  # (3, 384) numpy array
print(embeddings.shape)

# Cosine similarity
from sentence_transformers.util import cos_sim
print(cos_sim(embeddings[0:1], embeddings))
# tensor([[1.0000, 0.7234, 0.0856]])
```

Encoding is fast: ~5k texts/sec on a laptop CPU; 50k+/sec on a GPU.

For **batch encoding** of millions of items:

```python
embeddings = model.encode(texts, batch_size=64, show_progress_bar=True)
```

For **truly huge** workloads, the `sentence-transformers[onnx]` extras give 2-3× speedup, and OpenAI / Voyage commercial APIs scale to billions of vectors for a cost.

---

## Part 3: Storing embeddings as DataFrame columns

The cleanest pattern: **one embedding column per text column**.

```python
df = df.with_columns(
    title_embedding=pl.col("title").map_elements(
        lambda t: model.encode(t).tolist(),
        return_dtype=pl.List(pl.Float32),
    )
)
```

Storage:

| Format | Notes |
|---|---|
| **Parquet** | Built-in support for list / array columns; the right default |
| **Arrow** | Same |
| **HDF5** | Common for ML; supports chunked reads |
| **Vector DB** (Pinecone, Weaviate, Chroma, Qdrant) | Specialized; supports ANN search at scale |
| **DuckDB with FLOAT[] column** | Embedded; great for laptop work |

For a tabular pipeline with up to ~10M rows: **Parquet with array columns**. Beyond that, a vector DB.

---

## Part 4: Approximate nearest neighbor (ANN) search

For "find the 10 most similar items in this 1M-item table", you can't do exhaustive cosine - that's 1M comparisons per query.

The fix: **ANN indexes** - data structures that return approximate top-k in O(log n) per query.

| Algorithm | Lib | Strength |
|---|---|---|
| **HNSW** (Hierarchical Navigable Small World) | hnswlib, FAISS-HNSW | Best quality at sub-millisecond latency |
| **IVF** (Inverted File) | FAISS-IVF | Good quality at million-scale |
| **IVFPQ** (IVF + Product Quantization) | FAISS | Compressed storage for billion-scale |
| **ANNOY** | annoy (Spotify) | Read-only; great for batch search |
| **DiskANN** | DiskANN | Disk-backed for huge corpora |

Common library: **FAISS** (Meta) for production; **hnswlib** for "I want HNSW in 20 lines."

```python
import hnswlib
import numpy as np

# Build index
dim = 384
index = hnswlib.Index(space="cosine", dim=dim)
index.init_index(max_elements=1_000_000, ef_construction=200, M=16)
index.add_items(embeddings, ids=np.arange(len(embeddings)))
index.set_ef(50)   # query-time accuracy / speed trade-off

# Search
query = model.encode("wireless earphones")
ids, distances = index.knn_query(query, k=5)
print(ids, distances)
```

Sub-millisecond queries on 1M items. **This is how RAG-style retrieval, semantic search, and recommendation systems work underneath.**

### Quality vs speed knobs

For HNSW:

- **`M`**: connectivity (more = higher quality, more memory; 16-32 typical)
- **`ef_construction`**: build-time quality (higher = better, slower build; 100-500)
- **`ef`**: query-time quality (higher = better recall, slower query; tune for your latency budget)

Default presets work for most use cases. Sweep `ef` against your eval set to find the recall/speed sweet spot.

---

## Part 5: Semantic deduplication

Embeddings make week 05's dedup vastly more powerful for cases where rapidfuzz misses.

The pattern:

```
1. Embed every record's key text column
2. For each pair: cosine similarity = candidate score
3. Threshold + cluster (same as week 05)
```

```python
embeddings = model.encode(df["product_name"].to_list())
# Use FAISS / hnswlib to find approximate near-neighbors
# For each item, get top-10 most similar; pairs with cos > 0.85 are candidate dupes
```

Compared to week 05's rapidfuzz:

| Approach | Catches | Misses |
|---|---|---|
| rapidfuzz | "Apple Inc." vs "Apple Inc" (typos, order) | "Apple Inc." vs "Apple Corporation" (semantic) |
| Embeddings | "Apple Inc." vs "Apple Corporation" (semantic) | "j0hn smitH" vs "John Smith" (heavy normalization needed) |

The right answer is often **both** - rapidfuzz for the typo cases, embeddings for the semantic cases.

---

## Part 6: Semantic classification

For "tag every support ticket by category" without labeled training data:

```
1. Embed each ticket
2. Embed each label as a short prototype ("billing issue", "shipping delay", "product defect")
3. For each ticket, assign the label with highest cosine similarity
```

This is **zero-shot classification**. Works surprisingly well for prototype categories that are linguistically distinct. Less well when categories overlap ("complaint" vs "frustration").

```python
labels = ["billing issue", "shipping delay", "product defect", "feature request"]
label_embs = model.encode(labels)

ticket_embs = model.encode(tickets)
similarities = cos_sim(ticket_embs, label_embs)
predicted = [labels[i] for i in similarities.argmax(dim=1)]
```

For better quality, fine-tune the embedding model on a small labeled set (a few hundred examples). For more flexibility (multiple labels, complex rules), use an LLM (Part 9).

---

## Part 7: Embedding-based retrieval (RAG basics)

The dominant use case in 2026: **retrieval-augmented generation**:

```
1. Embed all documents in your corpus → vector index
2. User asks question → embed the question
3. Search vector index for top-k most similar documents
4. Feed top-k as context to an LLM along with the question
5. LLM answers from the retrieved context
```

This week is the **retrieval** half. The LLM half lives in ml-mentorship. For data-engineering purposes:

- Build the vector index of your knowledge base (docs, tickets, code, anything)
- Make sure the index updates as the corpus changes (CDC pattern from week 03)
- Tune `k` and the similarity threshold against an eval set

Tools: **LangChain**, **LlamaIndex**, **Haystack** wrap this stack. For learning, build it from primitives once.

---

## Part 8: Multimodal embeddings (briefly)

For image / text joint embedding:

- **CLIP** (OpenAI) and **SigLIP** (Google) - text and images in the same vector space
- Search for "red sneakers" against an image catalog; the text query finds the right images

```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer("clip-ViT-B-32")

# Embed image and text in the same space
image_embs = model.encode([open_image("shoe.jpg")])
text_emb = model.encode(["red sneakers"])
print(cos_sim(text_emb, image_embs))
```

For e-commerce product search, visual search, content moderation. Beyond this curriculum's main thread, but a powerful tool.

---

## Part 9: Embed vs prompt - the modern decision

For "classify these support tickets," you have two options:

| Approach | Cost | Latency | Flexibility |
|---|---|---|---|
| **Embed + train classifier** | ~$0 in inference | <1ms | Fixed categories |
| **Embed + zero-shot** | ~$0 in inference | <1ms | Flexible categories; lower quality |
| **Prompt LLM** | ~$0.01 per call | 0.5-2s | Maximum flexibility; explanations |
| **Fine-tune LLM** | $100s-$1000s | 0.5-2s | Best quality; locked categories |

In 2026, the practical decision tree:

- **<10 labels, you have a few thousand examples** → embed + train a logistic regression. Cheapest, fastest, predictable.
- **<10 labels, no training data, prototypes are linguistically distinct** → zero-shot embed.
- **Many labels, structured output required** → LLM with structured outputs (JSON mode).
- **Critical reliability, you have budget** → fine-tune an LLM (ml-mentorship week 12).

The economics shift yearly as models get cheaper. Re-evaluate periodically.

---

## Part 10: Vector storage - when to use a vector DB

For laptop-scale work (<10M vectors), **Parquet + FAISS/hnswlib in-memory** is plenty.

When to graduate to a vector DB:

| Need | Tool |
|---|---|
| Hundreds of millions of vectors, online serving | Pinecone, Weaviate, Qdrant, Chroma |
| Filter alongside vector search (metadata filter) | Most modern DBs (Qdrant especially) |
| Multi-tenant or hosted infra | Pinecone, Vespa, Weaviate Cloud |
| Co-locate with Postgres | `pgvector` extension (production-grade now) |
| Co-locate with DuckDB / lakehouse | DuckDB VSS extension; LanceDB |

For most teams in 2026 starting from scratch: **pgvector** if your stack is already Postgres; **Qdrant** if greenfield; **LanceDB** if you want a Parquet-friendly local-first DB.

---

## Part 11: Embedding drift

A model's embeddings shift if you change the model. So:

- **Don't mix vectors from different models** in one index
- **When you upgrade the embedding model, re-embed the whole corpus**
- **Store the model name + version alongside embeddings** as metadata

This is the **embedding drift** problem. Without versioning, you'll have a multi-day debugging session when search starts returning weird results six months from now.

---

## Part 12: Anti-patterns

| Anti-pattern | Cost |
|---|---|
| Storing embeddings without model name/version | Drift detection becomes impossible |
| Using OpenAI embeddings in CI without caching | Bill explosion |
| Computing cosine similarity in a Python loop | 100-1000× slower than vectorized numpy / sklearn |
| Sending PII to a hosted embedding API | Compliance / contractual issues |
| Using a 3072-dim model when 384 would do | Storage cost; index size; query latency |
| Treating embeddings as deterministic across model versions | They aren't |

---

## Part 13: Connect to the rest of the curriculum

- **Week 05 (dedup)**: embeddings are the semantic-aware version of fuzzy matching
- **Week 08 (text)**: embedding models replace some text preprocessing; build on top of clean normalized text
- **Week 11 (features)**: embeddings as columns in feature pipelines
- **Week 16 (production)**: vector indexes as part of the production pipeline
- **ml-mentorship week 16**: the LLM serving side that closes the RAG loop

---

## What's next

In [lab.md](lab.md) you'll:

- Embed 50k product descriptions with sentence-transformers
- Compare CPU vs onnx vs OpenAI API speeds + cost
- Build an HNSW index; benchmark sub-millisecond search
- Implement semantic dedup; compare to week 05's rapidfuzz results
- Build zero-shot classification of NYC 311 tickets
- Store embeddings as a Polars array column + Parquet
- (Stretch) Set up pgvector and search against it

By end of week 12 you can embed any text, build a vector index, and ship semantic search / dedup / classification on tabular data.
