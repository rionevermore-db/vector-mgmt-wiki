---
query-key: scale-tier-shifts
query: "向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在索引选择、存储介质、并行能力这三个维度上，哪些决策在哪一档发生质变？"
date: 2026-05-12
phase: post
ingest-context: santhanam-2022-colbertv2
wiki-pages-total: 79
cited-pages: [concepts/colbertv2.md, topics/disk-vs-memory-ann.md]
cited-count: 2
---

# Post-snapshot (santhanam-2022-colbertv2): scale-tier-shifts

## TL;DR (delta from lancedb-docs post)

**ColBERTv2 引入 wiki tier-shift "multi-vector storage scaling penalty"**——late-interaction 单 doc 是 M (~128) token vectors, 在每个 scale tier 上 storage cost 是 single-vector 的 128× (减 residual compression 6-10×, 仍 ~13-20× net). 这让 ColBERTv2 在 high-tier (>1B docs) **storage bottleneck** 远早出现.

## Answer

### Tier-shift × multi-vector storage 新轴（NEW）

[per santhanam-2022-colbertv2 §3.3 + tier-shift]

| Tier | Single-vector storage (768-d int8 + RaBitQ) | **ColBERTv2 multi-vector (128 token × 20 bytes residual)** |
|---|---|---|
| ≤ 1B | ~32 GB | **~2.5 TB** |
| 1-10B | 320 GB | 25 TB |
| 10-100B | 3.2 TB | 256 TB |
| 100B-1T | 32 TB | 2.5 PB |
| ≥ 1T | 320 TB | 25 PB |

→ ColBERTv2 在每 tier 是 single-vector 的 ~80×. 实际 production: tier 3 (10-100B) 是 late-interaction 的 deployment ceiling—— ColBERTv2 千亿 production case **wiki + 论文均 zero coverage**.

### 决策表（updated 2026-05-12 post colbertv2）

- **Tier 1-2 (≤10B)**: ColBERTv2 viable + sparse + dense 3-way hybrid 全 active
- **Tier 3 (10-100B)**: ColBERTv2 storage tight, 需要 Vespa/Milvus dedicated infrastructure
- **Tier 4+ (>100B)**: late-interaction 不主流; sparse + dense 2-way 主导

### 已知盲区

- **千亿 ColBERTv2 production case**: 不存在
- **Tier 4 late-interaction crossover**: 何时切换到 sparse-dense-only? Unknown
- **BGE-M3 multi-vector at scale**: BGE-M3 把 colbert-style 作 single model 三 vector type 之一, large-scale 实测不公开

## Cited Pages

- [concepts/colbertv2.md](../../concepts/colbertv2.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
