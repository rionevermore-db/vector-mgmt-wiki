---
query-key: vector-scalar-bench-methodology
query: "对比多个 vector DB 在向量 + 标量过滤混合查询下的性能，如何公平地横向 benchmark？"
date: 2026-05-11
phase: post
ingest-context: kusupati-2022-matryoshka
wiki-pages-total: 73
cited-pages: [concepts/matryoshka-embedding.md, topics/adaptive-retrieval-shortlist-rerank.md, topics/attribute-filtering.md]
cited-count: 3
---

# Post-snapshot (kusupati-2022-matryoshka): vector-scalar-bench-methodology

## TL;DR (post-ingest of kusupati-2022-matryoshka)

**MRL 给 fair benchmark methodology 引入新维度: prefix dim 必须作为 benchmark axis**——production MRL-trained embedding 在不同 prefix dim 上 recall × latency 曲线**显著不同**, 公平比较必须 sweep prefix dim. **关键 NEW**: 之前 wiki fair benchmark 框架 (DistributedANN Table 1 范本) 假设 single d 维; MRL 引入**dim sweep + filter sweep + selectivity sweep 三维 benchmark matrix**. 同时, vendor-native MRL support (Vespa matryoshka cell type) vs application-level MRL (其他 vendor) 必须公平区分 — Vespa native 实测有 0 application overhead, 其他 vendor 必须 truncate at client side.

## Answer

### 与之前 ingest 的演进

| | radford-2021-clip post | **kusupati-2022-matryoshka post (NEW)** |
|---|---|---|
| Benchmark axis | scale, selectivity, recall, latency | **+ MRL prefix dim sweep** |
| 多 vendor benchmark | per-vendor implementation | **+ native MRL support vs application-level distinction** |
| Production MRL fair compare | n/a | **NEW: Vespa matryoshka cell vs application-level truncate distinction critical** |

### MRL-aware fair benchmark framework（updated 2026-05-11 post kusupati-2022-matryoshka）

[per kusupati-2022-matryoshka §4 + DistributedANN Table 1 范本 + wiki industry coverage]

**Tier 1: Capability inventory matrix**:
| Vendor | Native MRL? | Multi-dim cell type? | Application-level prefix support? |
|---|---|---|---|
| **Vespa** | **✓ matryoshka cell type** | **✓ tensor<float>(i{},x[N])** | ✓ (additionally) |
| Milvus | ✗ | ✗ (multi-vector field 各自 fixed dim) | ✓ |
| Qdrant | ✗ | ✗ | ✓ |
| Weaviate | ✗ | named vectors 但各自 fixed | ✓ |
| Pinecone | 不公开 | 不公开 | ✓ (client side) |
| Turbopuffer | ✗ | cell type fixed dim per namespace | ✓ via namespace-per-version |

**Tier 2: Benchmark protocol (3-axis sweep)**:
- **Axis 1: Selectivity** ∈ {0.01%, 0.1%, 1%, 10%, 50%, 90%, 99%}
- **Axis 2: MRL prefix dim** ∈ {64, 128, 256, 512, 1024} (or model-specific M)
- **Axis 3: Filter complexity** (simple Eq / complex AND-OR / range / glob/regex)

**Tier 3: Metrics**:
- Recall@5, recall@200 (at each prefix dim + selectivity)
- Latency p50 / p99 (at each prefix dim + selectivity)
- Throughput at each prefix dim
- Storage / memory per vector at each prefix dim
- **Native vs application overhead**: Vespa native vs Milvus application-level prefix MUST report separately

**Tier 4: Pitfalls**:
- 仅测 full dim → 忽略 MRL prefix advantage
- 仅测 prefix dim → 忽略 rerank stage cost
- 不区分 native vs application-level prefix support
- 用 non-MRL embedding (legacy CLIP / SIFT) 评估 → 不反映 production MRL embedding 性能

### Filter integration 与 MRL prefix 的正交性

[per topics/attribute-filtering.md + concepts/matryoshka-embedding.md]

Filter 与 MRL prefix 是**正交两 axis**:
- Filter: 限制 candidate set (selectivity)
- MRL prefix: 减少 per-candidate distance compute (dim cost)

公平 benchmark 必须**独立 sweep** 两 axis. e.g., Qdrant Filterable HNSW + MRL prefix 256-d 在 selectivity 0.1% vs Vespa Acorn-1 + matryoshka 256-d 在 selectivity 0.1% — head-to-head 不存在公开.

### 已知盲区

- **多 vendor MRL-aware fair benchmark**: 不存在公开
- **Vespa matryoshka cell type vs application-level MRL**: head-to-head 量化 unknown
- **MRL prefix × filter selectivity 联合实测**: wiki + vendor 都 zero coverage
- **Cross-vendor MRL benchmark standardization**: 不存在 (类似 ANN-Benchmarks 但 MRL-aware)

## Cited Pages

- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
- [topics/adaptive-retrieval-shortlist-rerank.md](../../topics/adaptive-retrieval-shortlist-rerank.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
