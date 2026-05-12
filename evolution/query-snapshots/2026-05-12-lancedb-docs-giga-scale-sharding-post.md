---
query-key: giga-scale-sharding
query: "当我面对千亿、万亿级向量规模，在私有云部署（16 个 128U + 1TB RAM 节点），采取何种分片策略构建索引？Workload 假设：read-heavy，目标 P99 < 50ms，向量维度 768，原始数据周级别全量更新一次。"
date: 2026-05-12
phase: post
ingest-context: lancedb-docs
wiki-pages-total: 78
cited-pages: [systems/lancedb.md, systems/milvus.md, systems/chroma.md]
cited-count: 3
---

# Post-snapshot (lancedb-docs): giga-scale-sharding

## TL;DR (delta from chroma-docs post)

**LanceDB Enterprise 进入 wiki 内 "petabyte-scale multimodal lakehouse" production category**——明示支持 petabyte-scale (video / point cloud / image + vector + metadata). **关键 NEW**: 与之前 wiki vendor (主要 text/embedding) 不同, LanceDB Enterprise 处理 **multimodal raw data + vector 同存** 的 production case. 千亿规模 sharding 走 Lance format columnar partitioning + distributed query engine (Enterprise only).

## Answer

### 与之前 ingest 的演进

| | chroma-docs post | **lancedb-docs post (NEW)** |
|---|---|---|
| Sharding 7 路径 (a)-(g) | unchanged | **不变** |
| Petabyte-scale multimodal production | 未涵盖 (CLIP-style metadata only) | **NEW: LanceDB Enterprise 明示 petabyte-scale multimodal (video / point cloud / image)** |
| Vendor 多模态 raw data + vector 同存 | 7 vendor 都 vector + metadata only | **+ LanceDB: vector + metadata + raw multimodal** |

### LanceDB Enterprise giga/petabyte-scale 哲学

[per sources/docs/lancedb/]

- **OSS embedded library**: dev / 中小规模 production (亿级 vectors single node)
- **LanceDB Enterprise**: distributed managed petabyte-scale multimodal lakehouse
- Lance format columnar partitioning 内置 (类似 Iceberg / Delta / Hudi)
- Distributed query engine (Enterprise) + Lance format core

vs 其他 vendor 千亿 paths:
- DistributedANN: single graph distributed via KV store (Bing-style)
- Turbopuffer: namespace-as-primitive (100M+ S3 prefix)
- Vespa: content cluster groups
- Milvus: cloud-native disaggregated segment
- Pinecone: slab adaptive
- Chroma: SPANN-based Cloud (scale不明示)
- pgvector: Citus distributed
- **LanceDB**: Lance format columnar partitioning + distributed query engine

### Petabyte-scale multimodal production unique need (NEW)

[per LanceDB Enterprise positioning]

LanceDB Enterprise 主流 production case:
- Video / point cloud / image dataset management (petabyte)
- Feature engineering at scale (ML pipeline integration)
- Training data prep for large models (vector + raw multimodal 同 query)

→ 其他 wiki vector DBMS 通常 vector + metadata, 不存 raw image / video binary. LanceDB 是**唯一明示 multimodal raw + vector 同 production scale** vendor.

### 决策表（updated 2026-05-12 post lancedb-docs）

| Workload | 推荐 |
|---|---|
| **Multimodal lakehouse + petabyte-scale + raw data + vector 同 query** | **LanceDB Enterprise** |
| 千亿 single corpus + 复杂 ranking | Vespa SPANN + 4-phase ranking |
| 千亿 single corpus + 6× throughput | DistributedANN (Bing) |
| 多租户 multimodal SaaS | Turbopuffer namespace |
| RAG dev experience + AI agent | Chroma OSS Core / Cloud |
| ≤100M docs + 已 Postgres | pgvector |
| 千亿 + 多 index_type | Milvus DISKANN |

### 已知盲区

- **LanceDB Enterprise petabyte-scale production case**: 公开 customer 不存在
- **Lance format columnar partitioning 大规模实测**: 不公开
- **vs Iceberg/Delta/Hudi + vector overlay path**: 是否 LanceDB 是 Iceberg-like 平行 OR 上层封装? 不明示
- **GPU index building 在 petabyte-scale**: build cost / time 不公开

## Cited Pages

- [systems/lancedb.md](../../systems/lancedb.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/chroma.md](../../systems/chroma.md)
