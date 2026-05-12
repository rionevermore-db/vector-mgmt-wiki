---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-12
phase: post
ingest-context: formal-2021-splade-v2
wiki-pages-total: 75
cited-pages: [concepts/hnsw.md, concepts/splade-sparse-retrieval.md, topics/sparse-dense-hybrid-retrieval.md]
cited-count: 3
---

# Post-snapshot (formal-2021-splade-v2): hnsw-vs-nsg-selection

## TL;DR (delta from kusupati-2022-matryoshka post)

**SPLADE ingest 不改变 HNSW vs NSG dense-side 选择**——SPLADE 是 **sparse-side retrieval algorithm**, 不用 HNSW/NSG (proximity graph), 而用 **inverted index** (BM25-era infrastructure). **关键 NEW**: SPLADE 揭示 wiki vector DB 选择决策有**全新 axis**: **sparse + dense hybrid retrieval**——production retrieval 不是 "选 HNSW 还是 NSG" 而是 "**sparse path 选什么 (BM25 vs SPLADE) × dense path 选什么 (HNSW vs IVF)** × **fusion 怎么做**". HNSW 仍是 dense path default, 但**完整 production retrieval pipeline 不是单 HNSW**——总是 HNSW + sparse path 并存.

## Answer

### 与之前 ingest 的演进

| | kusupati-2022-matryoshka post | **formal-2021-splade-v2 post (NEW)** |
|---|---|---|
| HNSW production OSS DBMS | 4 (Milvus + Qdrant + Weaviate + Vespa) | **不变** |
| NSG production | 仅 Taobao 2B | **不变** |
| HNSW + dense vs HNSW + sparse path 视角 | HNSW + MRL prefix shortlist | **HNSW 是 dense path; SPLADE 是 sparse path; production hybrid 两路并存** |
| Sparse retrieval algorithm in wiki | BM25 mentions only, no source | **NEW: SPLADE 是 wiki 内首个 sparse neural retrieval algorithm** |

### Sparse retrieval ≠ HNSW/NSG (新 axis)

[per formal-2021-splade-v2 §3 + production reality]

SPLADE output 是 ~30,000-d sparse vector (大部分 0, few non-zero positions with weights). 不用 HNSW/NSG (dense ANN), 而用:
- **Inverted index** (BM25-era, posting list per term)
- 或 **sparse vector store** (Pinecone Sparse Index / Weaviate sparse-aware / Vespa weightedset)

→ HNSW vs NSG decision 仅 applies to **dense path** (CLIP/MRL embedding). Production 同时需 sparse path (SPLADE/BM25).

### Production retrieval pipeline 重新理解

[per formal-2021-splade-v2 + wiki 5 vendor hybrid + topics/sparse-dense-hybrid-retrieval.md]

之前 wiki 默认 production retrieval = ANN over dense embedding (HNSW + CLIP/MRL). **SPLADE ingest 后**:
```
Production pipeline =
  (sparse path: BM25 inverted index OR SPLADE inverted index)
  +
  (dense path: HNSW over CLIP/MRL embedding)
  +
  (fusion: α-blend / RRF / rank-profile)
  +
  (optional: cross-encoder rerank)
```

→ HNSW 是 **dense path** 选择 (vs NSG); 整体 retrieval pipeline 还需 sparse path 选择 (BM25 vs SPLADE) + fusion strategy.

### 选择决策表（updated 2026-05-12 post formal-2021-splade-v2）

**Dense path (HNSW vs NSG)**:
- HNSW 仍 wiki 5 vendor production default (NSG 仅学术)
- HNSW + MRL prefix shortlist 是 dense Adaptive Retrieval 默认
- HNSW + CLIP/MRL embedding 是 multimodal default

**Sparse path (BM25 vs SPLADE)**:
- BM25 (BlockMaxWAND): Weaviate / Vespa / Turbopuffer 当前 default
- SPLADE: 替代 BM25 next-gen sparse, MS MARCO 与 dense SOTA 持平, BEIR 11/14 best
- Production migration BM25 → SPLADE: 不公开

**Hybrid fusion (production-stack 4 种)**:
- Vespa rank-profile (first-class, 任意 expression)
- Weaviate `hybrid(α)` (single-param)
- Pinecone Sparse-Dense Hybrid Index (黑盒)
- Turbopuffer multi_query + application RRF (推到应用层)

### 已知盲区

- **SPLADE 是否能 fit ANN index (HNSW-style)**: SPLADE output 是 sparse 30K-dim; HNSW 默认用 dense. 是否有 sparse-aware ANN (类似 HNSW for sparse vectors)? Open
- **NSG + sparse path**: NSG 已无 dense production case, sparse 更无
- **BlockMaxWAND BM25 vs SPLADE inverted index 性能对比**: production lookup cost head-to-head 不公开
- **GPU sparse retrieval**: 全 wiki 都假设 CPU + inverted index; GPU SPLADE / sparse ANN 不存在公开

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
