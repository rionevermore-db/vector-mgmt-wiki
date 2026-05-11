---
query-key: scale-tier-shifts
query: "向量规模 10亿→百亿→千亿→万亿，在索引选择/存储介质/并行能力三轴上哪些决策发生'质变'（必须换方案）？"
date: 2026-05-11
phase: post
ingest-context: qdrant-docs
wiki-pages-total: 65
cited-pages: [systems/qdrant.md, topics/in-place-vs-out-of-place-updates.md]
cited-count: 2
---

# Post-snapshot (qdrant-docs): scale-tier-shifts

## TL;DR (delta from ootomo-2023-cagra post)

**Qdrant 不引入新质变维度——但暴露第 4b sub-dimension (segment-level DBMS) 的 OSS 路径分化**：之前 4b 主要是 Milvus segment 模型（Go + 复杂 disaggregated）；Qdrant 是同 4b 维度的不同实现哲学（Rust 单 binary + shard 简洁路径）。**对 scale tier shift 的影响**：第 4 维度 (library → DBMS) 细化为 (4a single-server) / (4b segment-level disaggregated, Milvus) / (4c GPU memory-resident, CAGRA) / **(4d-NEW Rust 简洁 shard, Qdrant)** —— 同 segment-level scale 范畴但工程实现根本不同。

## Answer

### 12 个质变点 + sub-dimensions（updated）

| # | 质变点 | 维度 | 触发 |
|---|---|---|---|
| 1 | 索引必要性 | 规模 | ~10M |
| 2 | 单机 RAM 触顶 | 规模 + 存储 | ~1B |
| 3 | 单机 SSD 触顶 | 规模 + 存储 | ~百亿+ |
| **4** | **library → DBMS** | 工程形态 | dynamic + 分布式 + filter |
| 4a (sub) | single-server CPU+SSD DBMS | 部署假设 | DiskANN/SPANN single-server |
| 4b (sub) | segment-level cloud-native DBMS | 部署假设 | Milvus disaggregated |
| 4c (sub) | GPU memory-resident DBMS | 部署假设 | CAGRA / NVIDIA RAFT |
| **4d (NEW sub)** | **Rust 单 binary shard DBMS** | **部署假设** | **Qdrant 简洁路径** |
| 5 | out-of-place rebuild → in-place | update strategy | 持续 update + 资源约束 |
| 5a (sub) | cluster path (SPFresh/LIRE) | memory budget ≤ ~10 GB | cost-sensitive streaming |
| 5b (sub) | graph path (FreshDiskANN/FreshVamana) | memory budget ~128 GB | recall-sensitive streaming |
| 6 | static index → adaptive | 索引设计 | SaaS / vendor adaptive |
| 7 | strong/eventual → tunable τ | consistency | vector DB 多样化 |
| 8 | vector-first DBMS → DB-extended | 架构起点 | 8a OLAP / 8b OLTP |
| 9 | search-time filter → build-time filter-aware | filter selectivity | <25% selectivity |
| 10 | filter-aware build → predicate-agnostic | filter cardinality | >1000 filters |
| 11 | TopK interface → Iterator + RM | query 复杂度 | multi-column / range / Join |
| 11b | Biased PQ → Unbiased + sharp bound | quantizer 理论保证 | quantizer 层 K' 攻击 |

### 第 4d sub-dimension 特征（NEW）

[per systems/qdrant.md "与 Milvus 的对比" + sources/docs/qdrant/distributed_deployment.md]

| | 4b Milvus disaggregated | **4d Qdrant Rust shard** |
|---|---|---|
| 实现语言 | Go + C++ kernel | **Rust 单 binary** |
| 部署复杂度 | 4 层 (proxy/coordinator/query node/worker) | **单 binary 多 instance** |
| Cloud-native 设计 | Manu 4-layer disaggregated | **Raft consensus + shard 简洁** |
| Index 多样性 | 多 (HNSW/IVF*/DISKANN/CAGRA/SPARSE) | **HNSW only + Sparse** |
| Production engineering 主投入 | 多 index_type 集成 | **HNSW 工程深度** (Filterable HNSW + ACORN + Tenant index + 多 quantization) |
| Cluster scaling 复杂度 | Coordinator-managed | **手动 sharding (OSS) / auto rebalance (Cloud)** |
| Operational philosophy | "Tool box for all workloads" | **"Do HNSW exceptionally well"** |

→ **4b vs 4d 是 OSS vector DBMS 两条 production engineering 路径**——多 index_type 复杂适配 vs HNSW 单 index 深度工程。

### Qdrant 在 scale axis 上的位置（NEW）

| Scale | Qdrant 适配 |
|---|---|
| < 1M | 单 instance OSS / Cloud free tier |
| 1M - 100M | **Qdrant default sweet spot** (single instance / small cluster) |
| 100M - 1B | **Qdrant + 12 shards × 多 node** (production 实证) |
| 1B - 10B | Qdrant Cloud production；OSS 需手动 sharding + tuning |
| 10B - 千亿 | **不竞争**（不实证；推荐 Milvus / DiskANN / SPANN） |
| 千亿+ | 不竞争 |

→ Qdrant 在 1M-10B 范围是"production sweet spot"——千亿规模仍是 4b Milvus / 4a DiskANN-SPANN 主场。

### 与之前 ingest 的累积演进

| | gao-2024-rabitq post | wang-2024-starling post | singh-2021-freshdiskann post | ootomo-2023-cagra post | **qdrant-docs post (NEW)** |
|---|---|---|---|---|---|
| 质变维度数 | 11 (with 11b) | 11 (with 11b + 4a/4b) | 11 (with 4a/4b + 5a/5b) | 11 (with 4a/4b/4c + 5a/5b) | **12 (with 4a/4b/4c/4d + 5a/5b)** |
| Hardware path 维度 | 同前 | 同前 | 同前 | + 4c GPU | **+ 4d Rust 简洁** |
| OSS DBMS production engineering 路径 | 仅 Milvus 一种 | + Starling integration | + FreshDiskANN streaming | + CAGRA GPU | **+ Qdrant simple-Rust** |

### 不算质变（参数微调）

- Qdrant shard_number / replication_factor
- Qdrant HNSW m / ef_construct
- Qdrant quantization 选择
- Qdrant indexing_threshold / memmap_threshold

### 已知盲区

- **维度 4b vs 4d production benchmark**：Milvus vs Qdrant 千亿对比完全空白
- **Qdrant Hybrid Cloud 千亿实证**：不公开
- **Qdrant + iterator + RM (VBASE)**：理论可行未实证
- **维度 12+**：未来 ingest 是否暴露更多

## Cited Pages

- [systems/qdrant.md](../../systems/qdrant.md)
- [topics/in-place-vs-out-of-place-updates.md](../../topics/in-place-vs-out-of-place-updates.md)
