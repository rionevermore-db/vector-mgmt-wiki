---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-11
phase: post
ingest-context: qdrant-docs
wiki-pages-total: 65
cited-pages: [concepts/hnsw.md, concepts/nsg.md, systems/qdrant.md]
cited-count: 3
---

# Post-snapshot (qdrant-docs): hnsw-vs-nsg-selection

## TL;DR (delta from ootomo-2023-cagra post)

**Qdrant 是 HNSW 的纯粹押注**——Qdrant 整个 system 仅支持 **HNSW**（不像 Milvus 多 index_type），且大量工程投入在 HNSW 之上（Filterable HNSW extra edges + ACORN integration v1.16.0 + Tenant/Principal index + 多 quantization 变体）。这强化"HNSW 是 production vector DBMS 默认选择"的 wiki landscape——NSG 仍是论文 best-on-million-scale 但 production deployment 主要被 HNSW 占据（Pinecone 推断 HNSW + 黑盒、Milvus 多选项 default HNSW、Qdrant only HNSW）。

## Answer

### 与之前 ingest 的演进

| | ootomo-2023-cagra post | **qdrant-docs post (NEW)** |
|---|---|---|
| HNSW production 实证 | Pinecone (推断) / Milvus / VBASE / FreshDiskANN baseline / Faiss | **+ Qdrant 全押 HNSW** |
| NSG production 实证 | Taobao 2B (static)；未在主流 vector DBMS production 出现 | **不变**（Qdrant 不支持 NSG） |
| HNSW 工程演化深度 | + ACORN HNSW variant / VBASE HNSW iterator / CAGRA HNSW GPU-native | **+ Qdrant Filterable HNSW + ACORN integration + Tenant index** |
| Production HNSW 算法深度 | 多 variant 并存 | **Qdrant 是 HNSW production engineering 集大成者**（Filterable + ACORN + Tenant index + 多 quantization） |

### Qdrant 作为 HNSW production deployment 集大成者（NEW）

[per systems/qdrant.md + sources/docs/qdrant/manage-data/indexing.md]

Qdrant 在 HNSW 之上的工程：

| 工程层 | Qdrant 实现 |
|---|---|
| Base | HNSW (m + ef_construct + ef_search) |
| Filter-aware build | **Filterable HNSW**（per payload index extra edges） |
| Filter fallback | **ACORN algorithm** (v1.16.0) for strict filter combinations |
| Multitenancy | **Tenant Index** 优化 disk locality |
| Time-series | **Principal Index** 优化时间字段 |
| Quantization | Scalar / Binary / 1.5-2bit / Asymmetric / PQ - per-vector configurable |
| Sparse vectors | First-class with IDF (v1.10.0+) |
| Storage spectrum | in-memory / memmap / on-disk (HNSW + payload + vectors 都可独立配置) |
| Distributed | Raft consensus + sharding + replication_factor |
| Update model | LSM segment + WAL + Collection Aliases for migration |

→ NSG 完全缺乏类似 production engineering——纯学术算法 + Taobao 2B 部署仅 static。

### HNSW vs NSG 决策表（updated）

| 工程考量 | HNSW | NSG |
|---|---|---|
| 构建成本 in-memory | 高 | 更低 |
| Production engineering 深度 | **Qdrant + Milvus + Pinecone + Faiss + VBASE 全覆盖** | 仅 Taobao + NSSG (CAGRA paper baseline) |
| Filter-aware build | ✓ via ACORN / Qdrant Filterable HNSW | ✗ |
| Streaming insert+delete | ✗ (α=1 fail; FreshVamana α=1.2 work in graph-path) | ✗ |
| GPU-native | ✓ via CAGRA | ✗ (CAGRA paper 实测 Starling-NSG 替代等价 result) |
| OSS DBMS 集成 | **Milvus / Qdrant / Faiss 三家集成** | NSSG / Taobao only |
| Disk-resident segment | ✓ via Starling-HNSW | ✓ via Starling-NSG |
| Iterator + RM | ✓ via VBASE | ✗ |

### 选择决策（updated 2026-05-11）

- **CPU + 简单 single-vector TopK + million scale + 静态 + 内存富余 + 学术性能上限** → NSG
- **Production vector DBMS deployment** → **HNSW** (Qdrant / Milvus / Pinecone / Faiss / VBASE 都用)
- **OSS Rust + 简洁路径 + 仅 HNSW + filter heavy** → **Qdrant**
- **OSS Go + 多 index_type + complex workload** → Milvus
- **闭源 SaaS** → Pinecone

NSG 仅在"专攻 million-scale single-vector TopK 学术 benchmark"小众场景胜出。

### 已知盲区

- **NSG-based DBMS production**：Qdrant / Milvus / Pinecone 全选 HNSW；NSG production DBMS 缺
- **HNSW + Qdrant Filterable HNSW + RaBitQ 集成**：理论可行未实证
- **HNSW + iterator + RM (VBASE)**：HNSW base only，Qdrant 未集成 VBASE iterator
- **HNSW + CAGRA GPU + Qdrant Cloud**：Qdrant 当前 CPU only，GPU 集成 open

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [systems/qdrant.md](../../systems/qdrant.md)
