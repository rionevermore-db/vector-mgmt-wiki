---
query-key: embedding-update-handling
query: "当 embedding model 升级（比如从 BERT 换到 SBERT，或 OpenAI embedding v2 → v3）时，已有的向量索引和原始文档应该怎么处理？是否必须全量重建？有没有跨模型兼容/复用的方案？"
date: 2026-05-12
phase: post
ingest-context: muennighoff-2023-mteb
wiki-pages-total: 80
cited-pages: [benchmarks/mteb-massive-text-embedding-benchmark.md, concepts/clip.md, concepts/matryoshka-embedding.md]
cited-count: 3
---

# Post-snapshot (muennighoff-2023-mteb): embedding-update-handling

## TL;DR (delta from colbertv2 post)

**MTEB 是 wiki 内**首个客观 framework 评估 "embedding model 升级是否值得"**——upgrade cost 是 re-embed + 重建 index (具体 cost 与 vendor migration primitive 相关, per pgvector / Chroma / Turbopuffer / LanceDB / Vespa 等), upgrade value 由 MTEB score gap 量化. **关键 NEW**: MTEB leaderboard gap 5% 通常**不值得**升级 (考虑 migration cost); gap >10% 通常值得; 中间值需 task-specific 决策.

## Answer

### MTEB 作 embedding upgrade decision framework

[per muennighoff-2023-mteb + production reality]

**Upgrade decision matrix**:
| MTEB score gap | Decision (typical) |
|---|---|
| <2% | not worth upgrade |
| 2-5% | task-specific (some upgrade, some don't) |
| 5-10% | often worth, plan migration carefully |
| >10% | usually worth, prioritize migration |

**Embedding upgrade cost 与 vendor migration primitive 关系**:
- Lowest cost: pgvector ACID transaction row-level OR LanceDB zero-copy column-level
- Medium cost: Chroma CoW fork OR Qdrant aliases
- Higher cost: full re-embed (Vespa AppPkg cluster-wide / Turbopuffer namespace per version)

→ **MTEB gap + vendor migration cost** 两 axis 共同决定 upgrade decision.

### MTEB-evaluated production embeddings (current)

[per MTEB leaderboard + production reality]

- OpenAI text-embedding-3-large (3072-d MRL): commercial flagship
- Cohere embed-v3 / v4: multilingual commercial
- Voyage voyage-3 / multimodal-3: finance/legal specialized
- BGE-M3 (BAAI): OSS 三 vector type 一体
- NV-Embed-v2 / Linq-Embed-Mistral / SFR-Embedding-Mistral: top MTEB scores

### Algorithm 层 cross-model preserve frontier 仍未关闭

MTEB 衡量 individual model quality, **不衡量** cross-model embedding 是否在 cosine space 可 mapped. 跨 model migration 仍需 re-embed; talk SIGMOD 2026 live demo 是唯一 algorithm-level candidate.

## Cited Pages

- [benchmarks/mteb-massive-text-embedding-benchmark.md](../../benchmarks/mteb-massive-text-embedding-benchmark.md)
- [concepts/clip.md](../../concepts/clip.md)
- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
