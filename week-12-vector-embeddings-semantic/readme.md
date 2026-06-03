# Week 12: Vector Embeddings and Semantic Enrichment

## 🎯 What you'll learn

Use modern embedding models — **sentence-transformers**, **OpenAI / Voyage / Cohere embeddings**, **CLIP for images** — as a new kind of column in your data. The data-engineering side of the LLM era.

By the end of this week you'll be able to:

- Generate embeddings for text, image, and tabular fields at scale
- Build a small **vector index** (FAISS, HNSW via hnswlib) for sub-millisecond similarity search
- Do **semantic dedup**: "these two product descriptions mean the same thing"
- Do **semantic classification**: "tag every 311 complaint by category using embeddings + cosine similarity to label prototypes"
- Pick between **embedding-then-classical-ML** and **LLM-prompt-the-thing**
- Reason about cost: $/1M tokens for embeddings, vector storage per row

## 🧰 Lab setup

```bash
uv add sentence-transformers hnswlib chromadb pillow
```

## ✅ Your job

1. Read [theory.md](theory.md). The "embed vs prompt" decision section is the senior framing.
2. Work through [lab.md](lab.md). Embed 100k product descriptions; cluster them; semantically dedup; build a search index.
3. Build a "fuzzy dedup using embeddings" pipeline and compare to your week-05 rapidfuzz pipeline.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Sentence-Transformers docs](https://www.sbert.net/) | The default open embedding library | 30 min |
| [MTEB leaderboard](https://huggingface.co/spaces/mteb/leaderboard) | Which embedding model is best for what | 20 min |
| [Pinecone — Vector index basics](https://www.pinecone.io/learn/vector-database/) | The mental model (vendor-specific but well-written) | 30 min |
| [FAISS wiki](https://github.com/facebookresearch/faiss/wiki) | The library the field is built on | 30 min skim |

## 💡 What you should already know

- Weeks 01-11
- Optional: [ml-mentorship Week 10 (transformers)](https://github.com/ichdamola/ml-mentorship/tree/main/week-10-transformers) for what's under the hood

---

> 🚧 **Scaffolded.** Theory + lab fully fleshed in the next pass.

**Next**: [Week 13: EDA + Statistical Foundations →](../week-13-eda-statistical-foundations/readme.md)
