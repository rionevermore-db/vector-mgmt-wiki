---
title: 混合检索 benchmark 全景——vector+scalar / +full-text / +spatial 各自的成熟度
type: query
sources: [vectordbbench-docs, bigann-benchmarks-docs, thakur-2021-beir, muennighoff-2023-mteb, ann-benchmarks-docs]
related: [../benchmarks/vectordbbench.md, ../benchmarks/big-ann-benchmarks.md, ../benchmarks/beir-heterogeneous-zero-shot-ir.md, ../benchmarks/mteb-massive-text-embedding-benchmark.md, ../topics/ann-benchmarking-methodology.md, ../topics/attribute-filtering.md, ../topics/sparse-dense-hybrid-retrieval.md, ../topics/multimodal-embedding-retrieval.md, ../concepts/range-filter-ann-2024.md]
created: 2026-05-25
updated: 2026-05-25
---

# 混合检索 benchmark 全景

**Date**: 2026-05-25

**Question**:
有哪些比较通用的混合检索 benchmark?分别看 **vector + scalar(属性/数值过滤)**、**vector + full-text(BM25/稀疏)**、**vector + spatial(geo)** 三种,各自有没有公认 benchmark、成熟度如何?

## TL;DR

三种混合检索的 benchmark 成熟度**差异极大**:
- **vector + full-text**:**质量层成熟**——[BEIR](../benchmarks/beir-heterogeneous-zero-shot-ir.md)(主力)+ [MTEB](../benchmarks/mteb-massive-text-embedding-benchmark.md) + [big-ann Sparse track](../benchmarks/big-ann-benchmarks.md) + MS MARCO 底座;唯一薄的是**系统级融合性能横测**。
- **vector + scalar**:**中等成熟**——有 system-level case([VectorDBBench](../benchmarks/vectordbbench.md) Filtering)+ 竞赛 track(big-ann 2023 Filter)+ 算法 benchmark(ACORN / Filtered-DiskANN / range-filter)+ 5-strategy 方法论框架。
- **vector + spatial**:**benchmark 整体空白**——三大通用 ANN benchmark 全无 spatial track(注:spatial **index 能力** 已有 Vespa + LanceDB R-Tree 两家,但**仍无公平 benchmark**;capability ≠ benchmark)。

**两个真缺口**:vector+spatial(完全空白)+ vector+full-text 的**系统级 fusion 性能横测**。

## Answer

### 1. vector + scalar(属性 / 数值过滤)—— 中等成熟

[per topics/attribute-filtering.md, benchmarks/vectordbbench.md, benchmarks/big-ann-benchmarks.md]

- **系统级**:VectorDBBench 的 **Filtering case(int-based + label-based)**,30+ vendor 横测。
- **竞赛级**:big-ann 2023 **Filter track**(YFCC 10M + tag,FAISS baseline 3,200 QPS,标准化硬件 + 私有 query set)。
- **算法级**:[benchmarks/acorn-vs-filtered-diskann-nhq-milvus.md] + [benchmarks/filtered-diskann-vs-milvus-faiss-nhq.md];数值 range filter 有 [concepts/range-filter-ann-2024.md](../concepts/range-filter-ann-2024.md)(SeRF / iRangeGraph / UNIFY)。
- **方法论框架**:[topics/attribute-filtering.md](../topics/attribute-filtering.md) 5 策略(pre/post/hybrid)+ selectivity sweep。
- 缺口:跨 vendor 在**同一 selectivity 曲线**上的公平横测仍有限(vendor 用哪种 strategy 常不透明)。

### 2. vector + full-text(BM25 / 稀疏)—— 质量层成熟,系统层缺口

[per benchmarks/beir-heterogeneous-zero-shot-ir.md, benchmarks/mteb-massive-text-embedding-benchmark.md, benchmarks/big-ann-benchmarks.md]

**质量层(有标准 benchmark)**:
- **BEIR ← 主力**:18 datasets zero-shot,把 lexical(BM25/TF-IDF)+ dense + sparse(SPLADE)+ late-interaction(ColBERT)+ hybrid(BM25+CE)同框比。关键数:BM25 nDCG@10 **0.412 且 18/18 robust**;DPR 0.327(**低于 BM25**);**BM25+CE 0.467 = best hybrid**(算力贵);SPLADE v2 0.464。
- **MTEB**:embedding model 的 retrieval 质量(选模型)。
- **big-ann 2023 Sparse track**:neural sparse(MSMARCO/SPLADE)8.8M 规模,QPS@90% recall。
- **MS MARCO**:passage ranking 公共底座(MRR@10)。

**核心结论**:BM25 是 OOD robust baseline,dense 跨 domain 常打不过 BM25;production 共识 = sparse + dense + cross-encoder rerank [per topics/sparse-dense-hybrid-retrieval.md]。

**系统层缺口**:BM25+vector+RRF 融合的 **QPS/latency head-to-head 横测 ≈ zero**——BEIR 自承 "hybrid fusion 不在 single benchmark scope" [per benchmarks/beir-heterogeneous-zero-shot-ir.md];VDBBench 公开 case 无 hybrid fusion case;[topics/sparse-dense-hybrid-retrieval.md] Open Q 明记 "hybrid 公平 benchmark methodology zero coverage"。

### 3. vector + spatial(geo)—— 整体空白

[per topics/ann-benchmarking-methodology.md, topics/multimodal-embedding-retrieval.md, benchmarks/big-ann-benchmarks.md]

- **三大通用 ANN benchmark 全无 spatial track**:ann-benchmarks / VectorDBBench(filter 是 int/label)/ big-ann(两届均未设)。
- **根因**:能力**严重不对称**(真 spatial index / bbox 近似 / 完全没有)+ 算法**无共识**(geohash / R-tree / bbox-as-attribute)[per topics/multimodal-embedding-retrieval.md "vs Spatial Retrieval"]。
- **仅有的相邻拼图**:**原生 spatial index 现 2 家**——[Vespa](../systems/vespa.md)(position dimension)+ [LanceDB](../systems/lancedb.md)(**R-Tree,2026-02 新增** [per systems/lancedb.md §H]);[SeRF 2D segment graph](../concepts/range-filter-ann-2024.md) 是 lat/lon 2D range 的算法 building block;**YFCC 自带 geo 元数据但 big-ann 只用了 tag 过滤**——现成底座没被用作 spatial track。**这些都是 capability/算法,不是 benchmark——vector+spatial 公平横测仍 0**。
- 业界也无 MTEB/BEIR 级别的 vector+spatial 标准 benchmark(相邻的 "spatial keyword search" 传统是 spatial+稀疏关键词,非 spatial+dense vector)。

### 成熟度总表

| 混合类型 | 质量层 benchmark | 系统层横测 | 成熟度 | 真缺口 |
|---|---|---|---|---|
| vector + scalar | ✓ 算法 + 竞赛 track | ✓ VDBBench Filtering | **中** | selectivity 曲线公平性 |
| vector + full-text | **✓✓ BEIR/MTEB/big-ann sparse** | ✗ fusion 性能 zero | **质量成熟 / 系统缺** | BM25+vector fusion QPS 横测 |
| vector + spatial | ✗ 无 | ✗ 无 | **benchmark 空白** | benchmark 整条线（index 已有 Vespa + LanceDB,benchmark 仍 0） |

## Cited Pages

- [benchmarks/beir-heterogeneous-zero-shot-ir.md](../benchmarks/beir-heterogeneous-zero-shot-ir.md) — vector+full-text 质量层主力 benchmark
- [benchmarks/mteb-massive-text-embedding-benchmark.md](../benchmarks/mteb-massive-text-embedding-benchmark.md) — embedding model retrieval 质量
- [benchmarks/vectordbbench.md](../benchmarks/vectordbbench.md) — vector+scalar 系统级 Filtering case
- [benchmarks/big-ann-benchmarks.md](../benchmarks/big-ann-benchmarks.md) — Filter + Sparse track(scalar + full-text 竞赛级)
- [topics/ann-benchmarking-methodology.md](../topics/ann-benchmarking-methodology.md) — 三抽象层 + spatial 空白 Open Q
- [topics/attribute-filtering.md](../topics/attribute-filtering.md) — vector+scalar 5 策略框架
- [topics/sparse-dense-hybrid-retrieval.md](../topics/sparse-dense-hybrid-retrieval.md) — fusion 策略 + 系统层 benchmark 缺口
- [topics/multimodal-embedding-retrieval.md](../topics/multimodal-embedding-retrieval.md) — spatial vs multimodal 不对称
- [concepts/range-filter-ann-2024.md](../concepts/range-filter-ann-2024.md) — 数值 range filter + 2D spatial building block

## Follow-up Questions

- **vector+spatial benchmark 是否有 2024-2026 新工作**:知识截止 2026-01,值得 WebSearch;若有合规来源是高价值 ingest(填 wiki 最大 benchmark 空白)。
- **系统级 hybrid fusion benchmark**:BM25+vector+RRF 跨 vendor QPS/latency 横测——业界是否已有(或可基于 BEIR 数据集 + vendor 部署自建)。
- **BEIR 语料规模表**:BEIR 页缺每数据集 corpus/query 规模(单集 ~3.6K 到 ~14.9M),可抓 BEIR 论文 Table 补 deepen。
