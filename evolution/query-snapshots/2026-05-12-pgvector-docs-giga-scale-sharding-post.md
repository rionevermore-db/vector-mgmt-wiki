---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-12
phase: post
ingest-context: pgvector-docs
wiki-pages-total: 76
cited-pages: [systems/pgvector.md, systems/pase.md, systems/vbase.md, systems/distributedann.md]
cited-count: 4
---

# Post-snapshot (pgvector-docs): giga-scale-sharding

## TL;DR (delta from formal-2021-splade-v2 post)

**pgvector 给 giga-scale workload 添加全新 limitation**——pgvector **不内置 distributed routing**, 千亿规模 distributed 部署依赖**Citus** (Postgres distributed extension) 或 **application-level sharding**. **关键 NEW**: pgvector 哲学是 "vector 是 Postgres 一等公民 type, distributed 是 Postgres 已有 problem 而不是 vector-specific". vs Milvus / Pinecone / Vespa native distributed sharding, pgvector **委托给 Postgres ecosystem 解决 distributed problem**. 这是 production trade-off: simpler architecture (no separate vector DBMS) vs lower native scale ceiling (≤100M docs sweet spot per industry assertion). Giga-scale workload 是 pgvector **不主推**的 segment.

## Answer

### 与之前 ingest 的演进

| | formal-2021-splade-v2 post | **pgvector-docs post (NEW)** |
|---|---|---|
| Sharding 5 路径 (a-e) | unchanged | **不变** |
| pgvector 在 giga-scale workload | 未涵盖 | **NEW: pgvector 不主推 giga-scale; ≤100M sweet spot** |
| Postgres-extension vector retrieval | PASE (internal) + VBASE (research) | **+ pgvector (industry main, but ≤100M only)** |

### pgvector giga-scale 局限性（NEW critical finding）

[per sources/docs/pgvector/README.md §Scaling + industry usage]

pgvector 的 scaling 路径:
- **Vertical scale**: 单 Postgres instance 32 TB per non-partitioned table
- **Replication**: PostgreSQL streaming replication (read replicas only, primary single)
- **Distributed**: Citus extension OR application-level sharding

vs giga-scale dedicated vector DBMS:
- Milvus: cloud-native disaggregated (streaming/query/data nodes)
- Pinecone: slab adaptive distributed
- Vespa: content cluster groups + container stateless
- Turbopuffer: 100M+ S3-prefix namespace + stateless compute
- DistributedANN: single graph distributed via KV store (Bing 50B per slice)

→ **pgvector deliberately not competing on giga-scale**——it competes on "已 deploying Postgres + 加 vector retrieval" segment.

### 16 节点 + 1TB RAM × 768-d production workload 推算

[per pgvector README + production scaling reality]

**Workload assumption**: 千亿 docs (100B), 768-d, P99 < 50ms

**pgvector approach (theoretical, NOT industry-validated)**:
- 16 nodes × Citus distributed sharding
- Each node ~6.25B docs × 768-d × 4 bytes = 19.2 TB/node — 超 32 TB limit per non-partitioned table OK if partitioned
- HNSW per shard: large RAM requirement for HNSW build (`maintenance_work_mem`)
- Cross-shard query: Citus query planner fanout

**Realistic verdict**: 千亿规模 pgvector **理论上 viable via Citus**, 但**production case 不公开**——industry assertion pgvector ≤100M sweet spot 强烈. Giga-scale 主流仍是 separate vector DBMS.

### 千亿/万亿决策表（updated 2026-05-12 post pgvector-docs）

| Workload | 推荐方案 |
|---|---|
| **千亿 single corpus + 复杂 ranking + ML rerank** | Vespa SPANN + 4-phase ranking |
| 千亿 single corpus + throughput priority | DistributedANN single-graph distributed (Bing) |
| 多租户 multimodal SaaS | Turbopuffer namespace-as-tenant |
| 千亿 + 多 index_type | Milvus DISKANN / HNSW |
| Billion-scale + 闭源 SaaS managed | Pinecone slab |
| **≤100M docs + 已部署 Postgres workload + 加 vector retrieval** | **pgvector (lowest deployment friction)** |
| 学术 / research PG-extension iterator + RM | VBASE |
| Ant Financial 内部 OLTP + vector | PASE |

### pgvector sweet spot vs separate DBMS production segment 划分（NEW）

[per industry surveys + pgvector deployment reality]

- pgvector segment: ≤100M docs, **已 deploying Postgres**, lowest operational complexity
- Separate vector DBMS segment: >100M docs, dedicated vector workload, OR multimodal-spatial-sparse-hybrid native

→ **wiki 内 vector DBMS production segment 二分**:
1. **Pgvector-style "Postgres extension"**: low friction, low scale ceiling
2. **Separate DBMS "purpose-built"**: high scale, high feature, separate infrastructure

### 已知盲区

- **pgvector + Citus production scale**: 千亿 docs 实测 case 不公开
- **pgvector scale ceiling specifics**: "≤100M sweet spot" 是 industry assertion 不是 paper-backed quantification
- **pgvector vs Milvus head-to-head at 10M-100M**: 不公开
- **VACUUM cost on production giga-scale HNSW**: VACUUM 频率 vs write throughput trade-off 不公开

## Cited Pages

- [systems/pgvector.md](../../systems/pgvector.md)
- [systems/pase.md](../../systems/pase.md)
- [systems/vbase.md](../../systems/vbase.md)
- [systems/distributedann.md](../../systems/distributedann.md)
