---
query-key: scale-tier-shifts
query: "向量规模从 10 亿 → 百亿 → 千亿 → 万亿，在索引选择、存储介质、并行能力这三个维度上，哪些决策在哪一档发生质变？"
date: 2026-05-12
phase: post
ingest-context: formal-2021-splade-v2
wiki-pages-total: 75
cited-pages: [concepts/splade-sparse-retrieval.md, topics/sparse-dense-hybrid-retrieval.md, topics/disk-vs-memory-ann.md]
cited-count: 3
---

# Post-snapshot (formal-2021-splade-v2): scale-tier-shifts

## TL;DR (delta from kusupati-2022-matryoshka post)

**SPLADE 不改变 tier-shift 表的 dense-side criteria**——sparse-side inverted index 在 giga/tera scale **线性扩展, 不存在 dense ANN 的 tier-shift 复杂性**. **关键 NEW**: hybrid retrieval pipeline 在 each tier 内总是有 sparse + dense 双 path; sparse path 跟 corpus size **线性** scale 但 dense path 在每个 tier 上有不同选择. **Tier 5 (≥1T) 内 sparse path = 主力 recall provider**, 因 dense ANN approx 在 ≥1T 下 recall 退化更严重, sparse path exact match 更可靠.

## Answer

### 与之前 ingest 的演进

| | kusupati-2022-matryoshka post | **formal-2021-splade-v2 post (NEW)** |
|---|---|---|
| Tier-shift dense criteria | scale × storage × parallel × MRL prefix | **不变** |
| Sparse-side scale | 未涵盖 | **NEW: sparse inverted index linearly scales, 无 ANN tier-shift 复杂性** |
| Hybrid balance at tier 5 (≥1T) | dense ANN 主力 | **+ sparse path 更可靠 (exact match), 重要性提升** |

### Sparse-side 在各 tier 的 scale-shift behavior

[per formal-2021-splade-v2 §4 + IR production reality]

| Tier | Dense path 行为 | Sparse path 行为 |
|---|---|---|
| ≤ 1B | HNSW + memory | inverted index in memory, simple posting list |
| 1-10B | HNSW + quantization OR MRL prefix | inverted index in memory + posting list compression |
| 10-100B | SPANN / DiskANN / DistributedANN-style | inverted index on SSD (Anserini/Pyserini-style) |
| 100B-1T | DistributedANN-style required | inverted index sharded on SSD, simple doc-ID hash |
| ≥ 1T | Bing/Turbopuffer-style only 2 production cases | **inverted index sharded, no fundamental tier-shift** |

→ **Sparse-side**: 无 tier-shift 拐点, **linearly scales** (storage + posting list lookup cost 都是线性).

→ **Dense-side**: 各 tier 都有不同最优方案 (per kusupati-2022-matryoshka post).

### Hybrid balance shift at large scale

[per formal-2021-splade-v2 + Phase 1-2]

**≤ 10B (Tier 1-2)**: Dense path dominate; sparse 主要补 exact match (long tail tokens)
- Hybrid α ≈ 0.7 dense / 0.3 sparse (Weaviate default)
- Cross-encoder rerank cost-effective

**10B-100B (Tier 3)**: Dense path 开始 ANN approx 退化; sparse path 重要性提升
- Hybrid α ≈ 0.5-0.6
- DistilSPLADE-max alone 已接近 dense SOTA, sparse 不仅是补充

**100B-1T (Tier 4)**: Dense ANN approx 强制更严, recall 上限 ~90%; sparse path 是 exact recall fallback
- Hybrid α ≈ 0.4-0.5
- SPLADE BEIR 11/14 best 在大 OOD corpus 上 sparse path 更可靠

**≥ 1T (Tier 5)**: 在 wiki 内仅 Bing DistributedANN + Turbopuffer 2 数据点
- **Sparse path 重要性 strong**——exact match 对 long-tail query 至关重要
- 但 sparse-side 1T scale production case wiki + 论文都 zero coverage

### Tier-shift 决策驱动（updated 2026-05-12 post formal-2021-splade-v2）

1. **Tier 1 (≤1B) hybrid**: HNSW + (BM25 OR SPLADE) memory; α ≈ 0.7
2. **Tier 2 (1-10B) hybrid**: HNSW + quantization + sparse memory
3. **Tier 3 (10-100B) hybrid**: SPANN / DiskANN + sparse SSD; sparse 重要性提升
4. **Tier 4 (100B-1T) hybrid**: DistributedANN dense + sparse SSD sharded; sparse exact match fallback
5. **Tier 5 (≥1T) hybrid**: 仅 2 production case dense; **sparse path 1T+ case 完全空白**

### 已知盲区

- **Sparse path 1T+ scale production case**: SPLADE / BM25 on >1T docs not documented
- **Hybrid α 在不同 tier 的最优值**: production 不公开
- **SPLADE inverted index 1T+ posting list 单 term 长度 vs FLOPS regularizer 的关系**: 不验证
- **Cross-encoder rerank cost 在 ≥1T**: 不公开

## Cited Pages

- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
- [topics/disk-vs-memory-ann.md](../../topics/disk-vs-memory-ann.md)
