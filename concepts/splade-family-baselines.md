---
title: SPLADE Family Baselines（doc2query-T5 / DeepImpact / COIL）
type: concept
sources: [nogueira-2019-doc2query, mallia-2021-deepimpact, gao-2021-coil, formal-2021-splade-v2]
related: [splade-sparse-retrieval.md, blockmaxwand-bm25.md, colbertv2.md, ../topics/sparse-dense-hybrid-retrieval.md]
created: 2026-05-12
updated: 2026-05-12
---

# SPLADE Family Baselines

**TL;DR**: 3 paper 共同构成 **sparse neural retrieval pre-SPLADE 演化轨迹**——SPLADE v2 paper [formal-2021-splade-v2] §2-4 显式 cite 三者作 baselines, 之前 wiki 缺 source 支撑. **3 paper 各自路线**: (1) **doc2query / doc2query-T5** (Nogueira & Lin 2019): **document expansion** via query generation——用 seq2seq (后用 T5) 给 doc 生成可能 query, 把 expansion terms 加入 BM25 inverted index, 不修改 inverted index/BM25 structure 而是改 doc itself; (2) **DeepImpact** (Mallia 2021 SIGIR): **per-term impact learning**——doc2query-T5 expansion + 学习 per-term impact 直接 store in inverted index, **first paper 学习 BM25 等价 sparse weights**; (3) **COIL** (Gao 2021 NAACL): **Contextualized Inverted List**——每 token 一 contextualized dense vector, 在 inverted list 内做 exact-match + contextualized score, 是 BM25 与 ColBERT 之间的 hybrid. **对 wiki 内 vector DBs 的核心价值**: (a) **完整 sparse retrieval 演化 trajectory**: BM25 (foundational, 1994) → BlockMaxWAND (optimization, 2011) → DeepCT (Dai 2019, BERT term weights, not in this bundle) → **doc2query-T5 (2019)** → **DeepImpact (2021)** → **COIL (2021)** → **SPLADE v1/v2 (2021)** → BMP (Block-Max Pruning, 2024) → BGE-M3 (3-way unified, 2024); (b) **3 paper 各自 production impact**: doc2query 是 SPLADE 的 document expansion 思想 ancestor; DeepImpact 是 SPLADE 直接前身 (impact learning); COIL 是 ColBERT 的 sparse 路径变种, BGE-M3 multi-vector path 受 COIL 影响. [nogueira-2019-doc2query + mallia-2021-deepimpact + gao-2021-coil]

## 3 paper 各自概述

### 1. doc2query / doc2query-T5 (Nogueira & Lin 2019)

[per nogueira-2019-doc2query]

**Problem**: BM25 vocabulary mismatch——relevant docs 不包含 query terms.

**Approach**: 
- 训 seq2seq model (original: transformer; later docTTTTTquery: T5)
- Input: passage; output: predicted queries that passage answers
- Append predicted queries to passage → expanded passage
- Standard BM25 over expanded passages

**Results**: MS MARCO MRR@10 improvement over BM25 ~2-3pp.

**关键 insight**: **不需要修改 inverted index / BM25**, 只改 doc 本身——production-deployable on existing BM25 infrastructure.

### 2. DeepImpact (Mallia 2021 SIGIR)

[per mallia-2021-deepimpact]

**Problem**: doc2query-T5 + BM25 仍用 BM25 weighting (TF-IDF), 不学习 optimal term weights.

**Approach**:
- doc2query-T5 expansion (前提)
- 学习 **per-term impact value** (单 scalar per term) via 监督 ranking loss (MS MARCO triples + BM25 hard negatives)
- Impact values 直接 store in inverted index posting list (replacing BM25 TF-IDF)
- Query 时 standard inverted index lookup, summing impacts

**Results**: MS MARCO MRR@10 = 0.326 (vs BM25 0.184, doc2query-T5 0.277, COIL 0.341, SPLADE v1 0.322).

**关键 insight**: **First paper learning BM25-equivalent sparse weights via neural training**——SPLADE 的直接 ancestor.

### 3. COIL (Gao 2021 NAACL)

[per gao-2021-coil]

**Problem**: Dense retrieval (DPR) lost exact-match capability; BM25 lost semantic capability. Late-interaction (ColBERT) 全 doc 多 vector 太重.

**Approach**:
- 每 token 一 contextualized dense vector (类 ColBERT)
- Inverted list **per token**: 仅 exact token match 才 dense compute
- Final score = BM25 term match × contextualized vector dot product

**Results**: MS MARCO MRR@10 = 0.341. COIL-tok (small) = 0.317.

**关键 insight**: **Contextualized inverted list**——保留 inverted index efficiency + 加 contextualized score. 是 ColBERT 简化版 + BM25 增强版.

## 3 paper 共同主题: sparse neural retrieval 演化

[per 3 paper + SPLADE / ColBERT context]

| Paper | Year | Core innovation | Key limitation |
|---|---|---|---|
| doc2query | 2019 | Document expansion via query generation | 仍 BM25 weighting |
| DeepImpact | 2021 | Per-term impact learning over expanded doc | 仍 single-pass impact (no expansion built-in) |
| COIL | 2021 | Contextualized inverted list with exact-match | Token-level dense vectors increase storage |
| **SPLADE v2** | 2021-2022 | MLM-based sparse term weight + expansion | (本 paper 已 ingest, Phase 3) |
| BMP (Block-Max Pruning) | 2024 | BlockMaxWAND extension to SPLADE | (mentioned in BlockMaxWAND ingest) |
| BGE-M3 | 2024 | Single model: sparse + dense + late-interaction | (Batch #3 ingest) |

→ wiki 内 sparse retrieval **演化轨迹完整 articulated** with source 支撑.

## 与 wiki 内 ingest 关系

### Sparse retrieval source chain 完整

| Source | Year | wiki ingest |
|---|---|---|
| BM25 (classical) | 1994 | Not as standalone paper; referenced via BlockMaxWAND |
| BlockMaxWAND | 2011 | ✓ Batch #4 |
| **doc2query** | 2019 | ✓ **Batch #11** (此 ingest) |
| **DeepImpact** | 2021 | ✓ **Batch #11** (此 ingest) |
| **COIL** | 2021 | ✓ **Batch #11** (此 ingest) |
| SPLADE v2 | 2021-2022 | ✓ Phase 3 |
| ColBERTv2 | 2022 (late-interaction not sparse but cousin) | ✓ Batch #1 |
| BGE-M3 | 2024 | ✓ Batch #3 |

→ Sparse retrieval **完整 evolution chain 现在有 source 支撑**.

### Production current state (per 3 paper 演化)

- doc2query-T5: **production-deployable** (changes only doc text, runs on standard Lucene)
- DeepImpact: production research-stage (production case 不公开广泛 adoption)
- COIL: **production research-stage**, ColBERT family 后继 (ColBERTv2 等) 更主流

→ 三 paper 主要 **research significance**, production current 主流是 SPLADE / ColBERTv2 / BGE-M3.

## Open Questions

- **doc2query-T5 vs SPLADE production cost / quality head-to-head**: 不公开 (SPLADE 通常 quality 更高 but compute cost 也更高)
- **DeepImpact production adoption**: 仅论文层, production case 不公开
- **COIL vs ColBERTv2 head-to-head**: COIL 是 ColBERT 简化版, ColBERTv2 是 ColBERT 改进版——两者 trade-off 不公开
- **doc2query expansion + SPLADE 联合 production case**: 理论上 SPLADE 已含 expansion (via MLM logits), 是否 doc2query 额外 expansion 有 incremental gain? Open
- **BGE-M3 multi-vector path 与 COIL 关系**: 不公开 ablation

Cited by: 待 query 引用
