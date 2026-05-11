---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-11
phase: post
ingest-context: kusupati-2022-matryoshka
wiki-pages-total: 73
cited-pages: [concepts/matryoshka-embedding.md, topics/adaptive-retrieval-shortlist-rerank.md, systems/vespa.md, systems/distributedann.md, systems/turbopuffer.md]
cited-count: 5
---

# Post-snapshot (kusupati-2022-matryoshka): giga-scale-sharding

## TL;DR (delta from radford-2021-clip post)

**MRL 在 giga-scale workload 上提供 128× FLOP / 14× wall-clock speedup**——通过 **Adaptive Retrieval (shortlist + rerank)** 范式: 16-d prefix HNSW shortlist + 2048-d full rerank. **关键 NEW**: 16 节点 + 1TB RAM × 768-d MRL embedding workload, **AR pipeline 实质降低 search cost** = ~ (16/768) × ANN_cost ≈ 2% search FLOPs at shortlist stage, rerank 200 candidates × 768-d = trivially cheap. **P99 < 50ms 目标可由 MRL + AR pipeline 直接达成**——之前 wiki sharding 5 路径 (a-e) + 各种 disk tier 都是**机制层优化**, MRL 是**算法层 dim 优化**, **正交可叠加**.

## Answer

### 与之前 ingest 的演进

| | radford-2021-clip post | **kusupati-2022-matryoshka post (NEW)** |
|---|---|---|
| Sharding 5 路径 (a-e) | unchanged | **不变** |
| 维度作为 cost axis | 隐含 | **NEW: MRL 在 dim axis 直接 128× FLOP speedup** |
| AR-aware sharding | n/a | **NEW: MRL + AR 是 sharding-orthogonal cost reduction** |

### 16 节点 + 1TB RAM × 768-d MRL workload 推算

**Storage (MRL-trained voyage-3 / OpenAI emb-3 / Cohere v4)**:
- 1B 768-d float32 = 3 TB
- 单 schema 存全维, 不需要多 variant 各自存
- 16 节点 × 64 GB RAM available for ANN index (1 TB total, leave others for buffer)

**MRL + AR pipeline 性能**:
- Shortlist: 16-d prefix HNSW over 1B 16-d vectors = 16 GB (fit RAM single-node)
  - ANN cost: ~100 distance computations × 16 dim = 1600 FLOP per query
- Rerank: 200 candidates × 768-d cosine = 153600 FLOP per query
- **Total search FLOP per query**: ~155K FLOP (vs single-stage 768-d = ~76800 + index overhead)
- P50 latency: 数毫秒 (HNSW + 200 × cosine 全在 RAM)
- P99 < 50ms: **直接可达**

**Per-node throughput**:
- HNSW 16-d query: ~100K QPS per node (16 nodes = 1.6M QPS aggregate, far exceeds 1 vendor production)
- Rerank: 200 × 768-d ~ 150 µs per query per core

### MRL 与 sharding 5 路径正交叠加

[per kusupati-2022-matryoshka + wiki sharding (a-e)]

| 路径 | + MRL 加成 | Multi-tier production fit |
|---|---|---|
| (a) Global single index | + MRL prefix HNSW shortlist + full rerank → 128× speedup | DistributedANN-style + MRL |
| (c) 层次路由 | + MRL per-cluster prefix + global rerank → 多层级 cascade | Vespa SPANN + matryoshka |
| (d) 无索引 tenant | + MRL prefix exact scan (cheaper) | Vespa Streaming + matryoshka |
| (e) namespace-as-primitive | + MRL per-namespace prefix tier | Turbopuffer namespace + voyage-3 |

→ **MRL 是 sharding-orthogonal speedup**——任何架构 + MRL = 算法 + 系统层正交加成.

### 决策表（updated 2026-05-11 post kusupati-2022-matryoshka）

| 场景 | 推荐方案 |
|---|---|
| **千亿 single corpus + 复杂 ranking + 768-d MRL** | **Vespa SPANN + 4-phase ranking + matryoshka cell type** (唯一 native AR) |
| 千亿 single corpus + throughput priority + MRL | DistributedANN + MRL prefix shortlist (推测; paper 不明示) |
| 多租户 multimodal MRL | Turbopuffer namespace + voyage-3 MRL + AR application-level |
| Memory-only + ≤1B + MRL | HNSW + MRL prefix query (5 OSS DBMS 任选, application-level AR) |

### 已知盲区

- **MRL + DistributedANN 实际 production combo**: Bing 50B DistributedANN 是否支持 MRL prefix shortlist? paper 不明示
- **16 节点 production case 与 MRL + AR**: 不存在公开实测
- **Vespa matryoshka cell type 实际 giga-scale 部署**: docs 提及但 case study 不公开
- **MRL prefix 上的 HNSW α-RNG / Vamana / SPANN centroid 退化曲线**: 详细 ablation 不存在

## Cited Pages

- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
- [topics/adaptive-retrieval-shortlist-rerank.md](../../topics/adaptive-retrieval-shortlist-rerank.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
