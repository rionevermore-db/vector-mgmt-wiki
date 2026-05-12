---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-12
phase: post
ingest-context: chroma-docs
wiki-pages-total: 77
cited-pages: [systems/chroma.md, systems/turbopuffer.md, systems/spann.md]
cited-count: 3
---

# Post-snapshot (chroma-docs): giga-scale-sharding

## TL;DR (delta from pgvector-docs post)

**Chroma Cloud + SPANN + object storage 是 Turbopuffer SPFresh path 的 alternative**——两者都 object storage primary + Rust engine, 但 ANN 算法不同 (SPANN vs SPFresh). 千亿规模 production case: Chroma Cloud docs 不明示 hard scale ceiling (与 Turbopuffer 不同, Turbopuffer 公开 3.5T+ docs / 100M+ namespaces). **关键 NEW**: SPANN production deployment 从 3 增到 **4** (Microsoft Bing 历史 + Vespa OSS + Turbopuffer SPFresh 衍生 + Chroma Cloud)——SPANN family 在 production 占主流 object-storage-primary path.

## Answer

### 与之前 ingest 的演进

| | pgvector-docs post | **chroma-docs post (NEW)** |
|---|---|---|
| Sharding 5 路径 (a-e) + (f) | 6 (含 pgvector general-RDBMS) | **不变** |
| Object storage primary vendors | Turbopuffer (3.5T+ public scale) | **+ Chroma Cloud (scale 不公开)** |
| SPANN production deployment | 3 | **4 (+ Chroma Cloud)** |
| 千亿规模 vendor 候选 | Vespa + Turbopuffer + Pinecone + Milvus + DistributedANN | **+ Chroma Cloud (scale claim 不公开)** |

### Chroma Cloud 在 sharding architecture 内位置

[per sources/docs/chroma/llms-full.txt §Cloud + §Distributed Chroma]

Chroma Cloud architecture 与 Turbopuffer (路径 e) 相似:
- Object storage primary (S3 / GCS)
- Rust execution engine
- SSD + memory cache
- > 90% recall production target
- **Multi-tenant + single-tenant + BYOC** 3 deployment options (vs Turbopuffer multi-tenant + BYOC)

但 ANN 算法不同:
- Turbopuffer: SPFresh (SPANN + LIRE incremental)
- Chroma: SPANN (chen-2021 baseline)

→ Chroma Cloud 是 **SPANN family** 内 object-storage path 的另一个 implementation——production scale 不明示 hard limit.

### Object-storage-primary vendor 对比（updated 2026-05-12 post chroma-docs）

| | Turbopuffer | Chroma Cloud | DistributedANN (Bing) |
|---|---|---|---|
| Object storage primary | ✓ S3 / GCS | ✓ S3 / GCS | KV store (not object storage) |
| ANN | SPFresh | SPANN | DiskANN distributed graph |
| Public scale claim | 3.5T+ docs / 100M+ namespaces | not publicly stated | hundreds of billions per slice |
| Engine | Rust | Rust | C++ (Microsoft internal) |
| OSS variant | n/a | **Chroma Core OSS (single-node, HNSW)** | n/a |
| Pricing | usage-based public rates | usage-based public rates | n/a |

→ 三者中, **Chroma 是 OSS-Cloud bridge 独有 — 其他 object-storage-primary 都是 closed**.

### 千亿规模决策表（updated 2026-05-12 post chroma-docs）

| Workload | 推荐 |
|---|---|
| 千亿 single corpus + 复杂 ranking + ML rerank | Vespa SPANN + 4-phase ranking |
| 千亿 single corpus + 6× throughput | DistributedANN (Bing-style) |
| 多租户 multimodal SaaS + cost-sensitive | Turbopuffer namespace-as-tenant |
| **RAG-dev-experience priority + AI agent (MCP) native + dev-to-prod migration smooth** | **Chroma OSS Core (dev) → Chroma Cloud SPANN (prod)** |
| 千亿 + 多 index_type | Milvus DISKANN / HNSW |
| Billion-scale + managed simplicity | Pinecone slab |
| ≤100M docs + 已 Postgres | pgvector |

### 已知盲区

- **Chroma Cloud scale ceiling production data**: 不公开 hard limit (vs Turbopuffer 3.5T+ explicit)
- **Chroma SPANN vs Turbopuffer SPFresh production head-to-head**: 不公开
- **Chroma OSS Core 到 Cloud migration cost**: 不公开实际 case study

## Cited Pages

- [systems/chroma.md](../../systems/chroma.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/spann.md](../../systems/spann.md)
