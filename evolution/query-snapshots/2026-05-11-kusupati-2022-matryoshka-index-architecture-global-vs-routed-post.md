---
query-key: index-architecture-global-vs-routed
query: "在千亿/万亿规模下，索引架构应选 (a) 全局单一索引 / (c) 层次路由结构？两者在查询延迟、构建成本、召回率、增量更新、运维复杂度上各有什么 trade-off？工业上千亿/万亿规模主流选哪种、为什么？"
date: 2026-05-11
phase: post
ingest-context: kusupati-2022-matryoshka
wiki-pages-total: 73
cited-pages: [concepts/matryoshka-embedding.md, topics/adaptive-retrieval-shortlist-rerank.md, systems/vespa.md, systems/distributedann.md]
cited-count: 4
---

# Post-snapshot (kusupati-2022-matryoshka): index-architecture-global-vs-routed

## TL;DR (delta from radford-2021-clip post)

**MRL 不改变 5 架构 (a-e) 选择**——MRL 是 embedding-side, 与 sharding 架构正交. **关键 NEW**: 但 MRL + Adaptive Retrieval pipeline **天然映射到 multi-stage routing**: shortlist stage (low-d ANN) + rerank stage (full-d exact). 这与 (c) 层次路由的 multi-stage routing 哲学**完全同构**——只是从 "data partitioning" 变为 "embedding granularity partitioning". **Vespa 4-phase ranking + matryoshka cell type** 是 wiki 内唯一**同时支持 data + dim multi-stage routing** 的系统.

## Answer

### 与之前 ingest 的演进

| | radford-2021-clip post | **kusupati-2022-matryoshka post (NEW)** |
|---|---|---|
| 5 架构 (a)-(e) | unchanged | **不变** |
| Multi-stage routing 含义 | data partitioning (P 个 cluster, 选 top-N) | **+ embedding granularity (MRL prefix → full) — 新一类 multi-stage** |
| Vespa 4-phase ranking + matryoshka | tensor framework only | **NEW: 唯一 native 支持 dim-granularity multi-stage + data partitioning 联合** |

### MRL 与 sharding 架构的正交性 (NEW)

[per kusupati-2022-matryoshka §4 + wiki sharding 5 paths]

MRL **添加新 axis: embedding-granularity routing**, 与已有架构正交:

| Sharding axis (data) | + MRL axis (embedding dim) |
|---|---|
| (a) Global single index | 全 corpus single HNSW + MRL prefix shortlist → full rerank |
| (c) Hierarchical routing | 多 cluster 路由 + MRL prefix per-cluster shortlist + full rerank |
| (d) No-index tenant scan | tenant partition + MRL prefix exact (cheaper than full exact) |
| (e) Namespace-as-primitive | namespace per tenant + MRL prefix per namespace |
| **MRL granularity (new axis)** | shortlist dim → rerank dim (multi-stage by dim) |

### Vespa 4-phase + matryoshka = 二维 multi-stage routing first articulated

[per kusupati-2022-matryoshka + systems/vespa.md]

Vespa 是 wiki 内**唯一同时支持 data + dim multi-stage routing native** 的系统:

```
Query → retrieval (HNSW/BM25/weakAnd) — 用 256-d MRL prefix (cheap)
     → first-phase (per-shard cheap score) — 仍 256-d
     → second-phase (per-shard expensive rerank) — full 1024-d
     → global-phase (stateless container ONNX) — cross-encoder rerank
```

- **Data axis**: per-shard → global (data partitioning)
- **Dim axis**: 256-d prefix → 1024-d full → cross-encoder model (embedding granularity)
- **两 axis 独立由 ranking-profile 配置, 共同决定 latency/quality trade-off**

其他 vendor:
- Milvus: data axis (multi-vector field) 但 dim axis 推到 application
- DistributedANN: head index 是 dim-axis multi-stage (in-mem 2.5B head + full graph) 但与 MRL 集成 unknown
- Turbopuffer: namespace partition 是 data axis, dim axis 推到 application + model side QAT

### 已知盲区

- **MRL + (a) DistributedANN 联合**: 论文不明示, Bing 是否引入 MRL?
- **多 stage dim routing 在 Vespa 之外的 vendor 推 native**: industry 趋势预测向 Vespa 收敛, 但未发生
- **MRL granularity 选择自动化**: query-time prefix dim selection optimizer 不存在
- **MRL + DistributedANN single-graph 兼容性**: paper 不涵盖

## Cited Pages

- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
- [topics/adaptive-retrieval-shortlist-rerank.md](../../topics/adaptive-retrieval-shortlist-rerank.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/distributedann.md](../../systems/distributedann.md)
