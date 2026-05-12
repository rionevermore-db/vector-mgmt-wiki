---
query-key: vector-scalar-bench-methodology
query: "对比多个 vector DB 在向量 + 标量过滤混合查询下的性能，如何公平地横向 benchmark？"
date: 2026-05-12
phase: post
ingest-context: santhanam-2022-colbertv2
wiki-pages-total: 79
cited-pages: [concepts/colbertv2.md, topics/sparse-dense-hybrid-retrieval.md]
cited-count: 2
---

# Post-snapshot (santhanam-2022-colbertv2): vector-scalar-bench-methodology

## TL;DR (delta from lancedb-docs post)

**ColBERTv2 paper §5-6 实质提供 wiki 内 hybrid retrieval fair benchmark methodology range 范本——BEIR 28 datasets + LoTTE 12 domain-specific tests 共 40+ datasets cover IR + OOD + long-tail**. **关键 NEW**: ColBERTv2 评估 vs DPR / SPLADE / TAS-B / BM25 多 baseline 是 wiki 内 fair benchmark **跨 retrieval algorithm** 最完整 example.

## Answer

### ColBERTv2 paper fair benchmark range 范本

[per santhanam-2022-colbertv2 §4-6]

**Benchmark suite**:
- MS MARCO Passage Ranking (in-domain)
- BEIR 18+ datasets (zero-shot OOD)
- LoTTE 12 domain-specific tests (新引入 long-tail)

**Baselines covered**:
- BM25 (sparse classical)
- DPR (dense single-vector classical)
- TAS-B (dense, balanced topic-aware sampling)
- SPLADE v2 (sparse neural)
- ColBERT v1 (late-interaction baseline)
- RocketQA (dense + distillation)

→ **ColBERTv2 paper 是 wiki 内 first cross-retrieval-algorithm fair benchmark** (sparse + dense + late-interaction all covered).

### Fair benchmark methodology insights

1. **Multi-benchmark suite required**: 单 dataset (MS MARCO) 不足以反映 OOD generalization
2. **Long-tail topic evaluation**: 现实 production workload often domain-specific, BEIR semantic-similarity tasks 不充分
3. **Quality-space-cost trade-off**: ColBERTv2 报告 quality (MRR/NDCG) + space (bytes/vector) + retrieval cost (FLOPS) 三轴
4. **Baselines from 3 paradigms**: sparse + dense + late-interaction 才能 fair compare

### 已知盲区

- **公开 cross-vendor benchmark using ColBERTv2 paper methodology**: 不存在
- **LoTTE benchmark vendor adoption**: 未广泛采用
- **3-paradigm comparison + filter integration**: ColBERTv2 paper 不涵盖 attribute filter integration

## Cited Pages

- [concepts/colbertv2.md](../../concepts/colbertv2.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
