---
query-key: vector-scalar-bench-methodology
query: "对比多个 vector DB 在向量 + 标量过滤混合查询下的性能，如何公平地横向 benchmark？"
date: 2026-05-12
phase: post
ingest-context: formal-2021-splade-v2
wiki-pages-total: 75
cited-pages: [concepts/splade-sparse-retrieval.md, topics/sparse-dense-hybrid-retrieval.md, topics/attribute-filtering.md]
cited-count: 3
---

# Post-snapshot (formal-2021-splade-v2): vector-scalar-bench-methodology

## TL;DR (post-ingest of formal-2021-splade-v2)

**SPLADE 给 fair benchmark methodology 加 sparse-dense hybrid axis**——production retrieval 不仅是 vector + scalar filter, 还有 sparse + dense + filter 三维 hybrid. **关键 NEW**: 公平 benchmark 必须区分 (a) 纯 dense ANN + filter, (b) 纯 sparse retrieval + filter, (c) hybrid sparse + dense + filter; (a)(b)(c) 各自有不同最优 vendor + 不同 selectivity 曲线. **BEIR benchmark** (Thakur 2021, SPLADE 引用 但 wiki 未 ingest) 是 IR 标准 zero-shot benchmark, 提供 dense vs sparse vs hybrid fair comparison 范本.

## Answer

### Wiki 当前 vendor filter 实现表（unchanged from prior posts）

8 vendor filter 实现 inventory 未变 (Milvus / Qdrant / Weaviate / Vespa / Pinecone / Turbopuffer / DistributedANN-implied / Faiss).

### SPLADE 引入的 sparse-dense hybrid benchmark axis (NEW)

[per formal-2021-splade-v2 §4 + topics/sparse-dense-hybrid-retrieval.md]

**Vector-scalar benchmark methodology 之前 implicit assumption**: vector = dense embedding (CLIP / MRL / BGE).

**SPLADE ingest 后必须 explicit**:
- **Dense + filter**: dense embedding + scalar attribute filter (之前框架)
- **Sparse + filter**: SPLADE / BM25 sparse term + scalar filter (新)
- **Hybrid + filter**: sparse + dense + filter (production reality)

**Fair benchmark protocol**:

| Axis | Levels |
|---|---|
| Retrieval mode | dense only / sparse only / hybrid (α-blend or RRF) |
| Selectivity | 0.01% / 0.1% / 1% / 10% / 50% / 90% / 99% |
| Filter complexity | simple Eq / complex AND-OR / range / glob-regex |
| Recall@k | 5 / 10 / 100 / 200 (each retrieval mode) |
| Latency p50 / p99 | per retrieval mode |

### BEIR benchmark (NEW reference)

[per formal-2021-splade-v2 §4 + Thakur 2021]

**BEIR (Benchmark for IR)** [Thakur 2021, arxiv 2104.08663] 是 IR 标准 zero-shot benchmark:
- 14+ datasets across diverse domains (medical / legal / 财经 / ArguAna / DBPedia / FEVER / etc.)
- NDCG@10 primary metric
- **不属于 ANN benchmark family** (BIGANN / DEEP / SIFT) — IR-focused
- SPLADE paper §4 detailed BEIR results table → **DistilSPLADE-max 11/14 best** (avg 0.506)

→ BEIR 是 wiki 内 **best candidate for fair sparse-dense-hybrid benchmark** — 论文 BEIR results table 可作 wiki 内 fair benchmark example. wiki 应 future-ingest BEIR (Thakur 2021).

### Multi-vendor hybrid benchmark gap

[per topics/sparse-dense-hybrid-retrieval.md + 5 vendor]

- Vespa rank-profile vs Weaviate `hybrid(α)` vs Turbopuffer multi_query + RRF vs Pinecone Sparse-Dense Hybrid vs Milvus multi-vector field — **head-to-head 不存在公开**
- 各 vendor BlockMaxWAND BM25 vs SPLADE production migration cost / quality 不公开
- 不同 fusion 策略 (α-blend / RRF / rank-profile expression) 公平比较 wiki + vendor 都 zero coverage

### 已知盲区（updated 2026-05-12 post formal-2021-splade-v2）

- **5 vendor hybrid head-to-head Table 1-style benchmark**: 不存在
- **BEIR benchmark on production vector DBs**: 论文 evaluate sparse/dense models, not vendors; vendor-端 BEIR run zero
- **SPLADE × cross-encoder rerank cost / quality breakdown**: 论文不展开
- **Hybrid α / RRF k 优化与 workload 关系**: 不公开

## Cited Pages

- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
