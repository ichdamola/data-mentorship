# Week 12: Lab - Embed, Index, Search

You'll embed 50k product descriptions, build an HNSW index, run semantic dedup against your week 05 results, and finish with zero-shot classification of support tickets.

## Setup

```bash
uv add sentence-transformers hnswlib chromadb polars numpy matplotlib
```

```python
from sentence_transformers import SentenceTransformer, util
import hnswlib
import polars as pl
import numpy as np
import time
import json
from pathlib import Path

Path("data/silver").mkdir(parents=True, exist_ok=True)
```

We'll use the small fast model unless noted:

```python
MODEL_NAME = "all-MiniLM-L6-v2"
model = SentenceTransformer(MODEL_NAME)
print(f"loaded {MODEL_NAME}, dim={model.get_sentence_embedding_dimension()}")
```

---

## Exercise 12.1 - Embed a small batch

```python
texts = [
    "Apple AirPods Pro Wireless Earbuds",
    "Apple wireless earbuds with active noise cancellation",
    "Samsung Galaxy Buds Pro",
    "Sony WH-1000XM5 over-ear headphones",
    "raincoat",
    "yellow umbrella for rainy days",
    "kitchen blender",
    "high-speed blender for smoothies",
]

embs = model.encode(texts, convert_to_numpy=True)
print(f"shape: {embs.shape}")   # (8, 384)

# Cosine similarity matrix
sims = util.cos_sim(embs, embs).numpy()
print("\ncosine similarity matrix (rounded):")
for i in range(len(texts)):
    print(f"{texts[i][:30]:<30s}  ", " ".join(f"{s:+.2f}" for s in sims[i]))
```

You should see:

- AirPods Pro ≈ wireless earbuds (0.7+) - different words, same meaning
- AirPods vs Galaxy Buds (0.5-0.6) - both wireless earbuds; brand differs
- raincoat ≈ yellow umbrella (0.4-0.5) - both rain gear, weaker
- blender ≈ smoothie blender (0.6+)

**These are the kinds of matches rapidfuzz would have completely missed.**

---

## Exercise 12.2 - Generate 50k synthetic products

```python
import random
random.seed(42)

categories = {
    "headphones": ["headphones", "earbuds", "earphones", "audio gear"],
    "smartphone": ["smartphone", "mobile phone", "phone"],
    "laptop": ["laptop", "notebook computer", "ultrabook"],
    "shoes": ["shoes", "sneakers", "trainers", "footwear"],
    "clothing": ["shirt", "jacket", "pants", "sweater"],
    "kitchen": ["blender", "toaster", "kettle", "microwave"],
    "book": ["book", "novel", "guide"],
    "toy": ["toy", "puzzle", "game"],
}
brands = ["Apple", "Samsung", "Sony", "Nike", "Adidas", "Bosch", "Penguin", "LEGO", "Generic", "Premium"]
adjectives = ["wireless", "high-quality", "professional", "lightweight", "durable", "premium", "compact", "vintage"]

product_descriptions = []
for i in range(50000):
    cat = random.choice(list(categories.keys()))
    syn = random.choice(categories[cat])
    brand = random.choice(brands)
    adj = random.choice(adjectives) if random.random() < 0.7 else ""
    desc = f"{brand} {adj} {syn} model #{random.randint(1, 9999)}".replace("  ", " ").strip()
    product_descriptions.append({"product_id": i, "description": desc, "true_category": cat})

products_df = pl.DataFrame(product_descriptions)
print(products_df.head())
print(f"total products: {len(products_df):,}")
```

---

## Exercise 12.3 - Batch embed at scale

```python
t0 = time.perf_counter()
embeddings = model.encode(
    products_df["description"].to_list(),
    batch_size=128,
    show_progress_bar=True,
    convert_to_numpy=True,
)
t_embed = time.perf_counter() - t0
print(f"embedded {len(embeddings):,} products in {t_embed:.1f}s")
print(f"throughput: {len(embeddings)/t_embed:.0f} embeddings/sec")
print(f"size: {embeddings.nbytes / 1024 / 1024:.1f} MB")
```

On a laptop CPU: ~2-5k embeddings/sec → ~10-25s for 50k. With a GPU: ~50k/sec → sub-second.

---

## Exercise 12.4 - Build HNSW index, benchmark

```python
dim = embeddings.shape[1]
index = hnswlib.Index(space="cosine", dim=dim)
index.init_index(max_elements=len(embeddings), ef_construction=200, M=16)

t0 = time.perf_counter()
index.add_items(embeddings, ids=np.arange(len(embeddings)))
t_build = time.perf_counter() - t0
print(f"built index in {t_build:.1f}s")

index.set_ef(50)   # query-time accuracy

# Query
def search(query: str, k: int = 5):
    q_emb = model.encode(query)
    ids, distances = index.knn_query(q_emb, k=k)
    return [
        {
            "id": int(ids[0][i]),
            "distance": float(distances[0][i]),
            "description": products_df.row(int(ids[0][i]), named=True)["description"],
        }
        for i in range(k)
    ]

# Test queries
for q in ["wireless headphones", "running shoes", "phone with great camera"]:
    print(f"\nQuery: {q}")
    for r in search(q, k=3):
        # hnswlib's `cosine` space returns distance = 1 - cos_sim.
        # Smaller distance = more similar. Theory weeks talk in cos_sim
        # (0.6 = related, 0.3 = unrelated); show both so the mental model
        # carries over.
        cos_sim = 1 - r['distance']
        print(f"  cos_sim={cos_sim:.3f}  (dist={r['distance']:.3f})  {r['description']}")
```

Sub-millisecond per query against 50k vectors. **This is the engine that powers semantic search.**

---

## Exercise 12.5 - Semantic dedup vs rapidfuzz

Compare semantic similarity dedup to week 05's rapidfuzz approach on the same products.

```python
from rapidfuzz import fuzz

# Pick 100 random products as "queries"
query_ids = random.sample(range(len(products_df)), 100)
query_descs = [products_df.row(qid, named=True)["description"] for qid in query_ids]

# Embedding-based matches
t0 = time.perf_counter()
embedding_matches = []
for qid, qdesc in zip(query_ids, query_descs):
    q_emb = model.encode(qdesc)
    ids, distances = index.knn_query(q_emb, k=5)
    for cand_id, dist in zip(ids[0], distances[0]):
        if cand_id == qid: continue
        embedding_matches.append({
            "query_id": qid, "query": qdesc,
            "match_id": int(cand_id),
            "match": products_df.row(int(cand_id), named=True)["description"],
            "distance": float(dist),
        })
t_emb = time.perf_counter() - t0
print(f"embedding matches: {len(embedding_matches)} in {t_emb:.1f}s")

# rapidfuzz matches
t0 = time.perf_counter()
all_descs = products_df["description"].to_list()
rapidfuzz_matches = []
for qid, qdesc in zip(query_ids, query_descs):
    scores = [(i, fuzz.WRatio(qdesc, d)) for i, d in enumerate(all_descs) if i != qid]
    top5 = sorted(scores, key=lambda x: x[1], reverse=True)[:5]
    for cand_id, score in top5:
        if score < 85: continue
        rapidfuzz_matches.append({
            "query_id": qid, "query": qdesc,
            "match_id": cand_id,
            "match": all_descs[cand_id],
            "score": score,
        })
t_fuzz = time.perf_counter() - t0
print(f"rapidfuzz matches: {len(rapidfuzz_matches)} in {t_fuzz:.1f}s")
```

Compare matches that *only* embedding found:

```python
emb_pairs = {(m["query_id"], m["match_id"]) for m in embedding_matches}
fuzz_pairs = {(m["query_id"], m["match_id"]) for m in rapidfuzz_matches}

only_embedding = emb_pairs - fuzz_pairs
print(f"\nMatches only found by embeddings: {len(only_embedding)}")

if only_embedding:
    sample = list(only_embedding)[:5]
    for q_id, m_id in sample:
        q = products_df.row(q_id, named=True)["description"]
        m = products_df.row(m_id, named=True)["description"]
        print(f"  '{q}'  ↔  '{m}'")
```

You should see embedding matching things rapidfuzz missed - synonyms, different brands of the same product type. **This is the semantic-vs-syntactic gap closing.**

---

## Exercise 12.6 - Zero-shot classification of tickets

Take a small set of support tickets; classify them by embedding similarity to label prototypes.

```python
labels = [
    "billing or payment issue",
    "shipping or delivery delay",
    "product defect or quality complaint",
    "feature request or product question",
    "account access or login problem",
]
label_embs = model.encode(labels, convert_to_numpy=True)

tickets = [
    "I was charged twice for my last order - please refund.",
    "My package was supposed to arrive yesterday and it's still not here.",
    "The product broke after one day of use, this is unacceptable.",
    "Can you add a feature to export my data?",
    "I forgot my password and can't log in.",
    "When will my order ship?",
    "Why is the colour different from what was shown?",
    "Where can I see my invoice?",
]
ticket_embs = model.encode(tickets, convert_to_numpy=True)

similarities = util.cos_sim(ticket_embs, label_embs).numpy()
predicted = similarities.argmax(axis=1)

for i, (ticket, pred_idx) in enumerate(zip(tickets, predicted)):
    print(f"{ticket}")
    print(f"  → {labels[pred_idx]}  (sim {similarities[i, pred_idx]:.3f})\n")
```

You should see reasonable predictions: refund → billing, delivery → shipping, defect → product, etc. **Zero training examples; only label prototypes.**

For production-grade classification you'd:
1. Hand-label ~500 tickets
2. Fine-tune the embedding model or train a small classifier on top
3. Evaluate against a held-out test set

But for a first MVP, zero-shot embedding gets you 70-80% accuracy in 10 lines.

---

## Exercise 12.7 - Store embeddings in Parquet

```python
# Add embeddings as a list column
embeddings_list = [emb.tolist() for emb in embeddings]
products_with_embs = products_df.with_columns(
    embedding=pl.Series("embedding", embeddings_list, dtype=pl.List(pl.Float32)),
    embedding_model=pl.lit(MODEL_NAME),
    embedding_version=pl.lit("2024-01-15"),
)

products_with_embs.write_parquet("data/silver/products_with_embeddings.parquet")

# Verify roundtrip
loaded = pl.read_parquet("data/silver/products_with_embeddings.parquet")
print(loaded.head(3))
print(f"size on disk: {Path('data/silver/products_with_embeddings.parquet').stat().st_size / 1024 / 1024:.1f} MB")
```

50k × 384 × 4 bytes ≈ **77 MB on disk for the embeddings alone**. The metadata (model name, version date) is critical - it's what saves you when you upgrade models in 6 months.

---

## Exercise 12.8 - Save HNSW index, reload

```python
# Persist for production / next session
index.save_index("data/silver/products.hnsw")

# Reload
index2 = hnswlib.Index(space="cosine", dim=dim)
index2.load_index("data/silver/products.hnsw", max_elements=len(embeddings))
index2.set_ef(50)

# Verify it still works
sample = index2.knn_query(model.encode("running sneakers"), k=3)
print(sample)
```

The index file is binary, compact, and can be loaded into a production service. For tiny labels (50k), the index is ~30MB.

---

## Exercise 12.9 (stretch) - Use Chroma as a higher-level vector DB

For a more production-shaped path, swap HNSW for a real vector DB:

```python
import chromadb
client = chromadb.PersistentClient(path="data/chroma")

# Chroma's default distance is L2 squared. The embeddings throughout this
# lab were trained for cosine similarity; if you let Chroma default to L2,
# the rankings will be silently wrong (magnitude leaks into the score).
# Always set the metric explicitly to match how the embeddings were trained.
collection = client.get_or_create_collection(
    name="products",
    metadata={"hnsw:space": "cosine"},
)

collection.add(
    embeddings=embeddings.tolist(),
    documents=products_df["description"].to_list(),
    metadatas=[
        {"category": row["true_category"], "model": MODEL_NAME}
        for row in products_df.iter_rows(named=True)
    ],
    ids=[str(i) for i in range(len(embeddings))],
)

# Query with metadata filter
results = collection.query(
    query_embeddings=[model.encode("wireless headphones").tolist()],
    n_results=5,
    where={"category": "headphones"},   # only return headphones-category items
)
for doc, dist in zip(results["documents"][0], results["distances"][0]):
    print(f"  dist={dist:.3f}  {doc}")
```

Chroma gives you metadata filtering, persistence, and a HTTP server out of the box. **For production semantic search, the right starting point.**

---

## Submission checklist

- [ ] Embedded 8 manual examples; semantic similarities make sense
- [ ] 50k product descriptions embedded in batch (~10-25s on CPU)
- [ ] HNSW index built; queries return relevant results sub-millisecond
- [ ] Semantic dedup finds matches rapidfuzz missed
- [ ] Zero-shot ticket classification works for 8 tickets
- [ ] Embeddings stored as Parquet array column with model+version metadata
- [ ] HNSW index persisted and reloaded
- [ ] (Stretch) Chroma vector DB with metadata filtering

---

## What you just did

You can take any text column, generate dense embeddings, index them for sub-millisecond search, do semantic dedup that closes the gap rapidfuzz can't, and run zero-shot classification with just label prototypes. Embeddings are now a column in your Parquet, with provenance metadata that survives model upgrades.

For the rest of the curriculum (weeks 13-16: insights and production), embeddings become another tool in the kit - useful for fuzzy joins, clustering, semantic dashboards, and as a feature for ML models.

Week 13 turns to insight generation: how to do EDA the right way, beyond `df.describe()`.

---

**Next**: [Week 13: EDA + Statistical Foundations →](../week-13-eda-statistical-foundations/readme.md)
