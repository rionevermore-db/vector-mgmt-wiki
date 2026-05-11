---
title: Qdrant（Rust 实现的开源 vector DBMS）
type: system
sources: [qdrant-docs, weaviate-docs, vespa-docs]
related: [faiss.md, milvus.md, pinecone.md, weaviate.md, spfresh.md, freshdiskann.md, cagra.md, vbase.md, analyticdb-v.md, pase.md, starling.md, ../concepts/hnsw.md, ../concepts/acorn.md, ../concepts/filtered-vamana.md, ../concepts/product-quantization.md, ../concepts/rabitq.md, ../concepts/relaxed-monotonicity.md, ../concepts/freshvamana.md, ../topics/attribute-filtering.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md, ../topics/in-place-vs-out-of-place-updates.md, ../topics/topk-vs-iterator-model.md, ../topics/gpu-vs-cpu-ann.md]
created: 2026-05-11
updated: 2026-05-11 (Vespa peer cross-link)
---

# Qdrant

**TL;DR**: Qdrant Solutions GmbH 在 2021 开源的 **Rust 实现 vector DBMS**（Apache-2.0），与 [Milvus](./milvus.md) (Go) / [Pinecone](./pinecone.md) (闭源 SaaS) 形成 **OSS Rust / OSS Go / 闭源 SaaS 三足**。Qdrant 的**核心独特性**：(1) **Filterable HNSW**——HNSW graph 加 payload-aware extra edges 实现 filter-aware build（与 [FilteredVamana](../concepts/filtered-vamana.md) Vamana base 路径并列），v1.16.0 集成 **[ACORN algorithm](../concepts/acorn.md)** 作为 fallback——是 wiki 内首次 ACORN production deployment 实证；(2) **Quantization 多变体**：Scalar / Binary / **1.5-bit / 2-bit / Asymmetric** / Product，覆盖比 [Milvus](./milvus.md) / [Faiss](./faiss.md) 当前 release 更广的 quantization landscape；(3) **Collection Aliases for model migration**——atomic alias swap 让"新旧 embedding model 共存"成为 production-supported workflow（wiki 内首个明确 production migration tool，虽不解决跨 model algorithm 问题）；(4) **Tenant / Principal Index**——payload-aware storage layout，多租户 + 时间序列优化。**索引限制**：**仅 HNSW**（不支持 IVF / Flat / DiskANN 等其他 vector index——与 Milvus 多 index_type 生态对比）。Cloud SKU 提供 Resharding (v1.13.0+) 与 Hybrid Cloud / Private Cloud / Edge 部署。[per sources/docs/qdrant/]

## 与 wiki 现有系统的定位差异

[per qdrant-docs + wang-2021-milvus Table 1]

| | [Milvus](./milvus.md) | [Pinecone](./pinecone.md) | [Faiss](./faiss.md) | **Qdrant** |
|---|---|---|---|---|
| 实现语言 | **Go**（Go 团队 Zilliz） | 闭源（Rust assumed） | **C++17** | **Rust** |
| 形态 | Open Source DBMS + Zilliz Cloud | 闭源 SaaS only | Library (not DBMS) | **Open Source DBMS + Qdrant Cloud / Hybrid / Private / Edge** |
| 部署 SKU 数 | OSS + Zilliz Cloud | SaaS only | Self-host library | **OSS + Cloud + Hybrid + Private + Edge** |
| Index 多样性 | **多**：HNSW/IVF_FLAT/IVF_PQ/IVF_SQ8/DISKANN/SCANN/CAGRA/SPARSE | 黑盒 adaptive | 决策树多种 | **仅 HNSW** + Sparse |
| Filter-aware build | 5 strategy partition-based | metadata filter (黑盒) | IDSelector callback | **Filterable HNSW + ACORN integration** |
| Quantization 覆盖 | PQ / SQ8 (via Faiss) + ScaNN | 黑盒 | PQ family + ScaNN | **Scalar / Binary / 1.5/2-bit / Asymmetric / PQ** |
| Multitenancy | partition + RBAC | namespace | n/a | **payload partition + Tenant Index + Principal Index** |
| Model migration | n/a | n/a | n/a | **Collection Aliases (atomic switch)** |
| Sharding | segment + LSM | slab adaptive | n/a | **Raft consensus + consistent hashing + user-defined sharding** |
| Streaming insert | LSM segment growth | Pinecone serverless slabs LSM | n/a | **WAL + segment optimizer**（vs FreshDiskANN 的 streaming graph） |
| 商业体量 | 300+ enterprise (Salesforce, NVIDIA 等) | 闭源 SaaS 客户 | Meta + Zilliz + 工业生态 | OSS 用户 + Qdrant Cloud (Stripe, AT&T 等) |

**核心论点**：Qdrant 在 wiki 内**填补 Rust 开源 vector DBMS** 位置——简洁单 binary + HNSW 单 index 路线 vs Milvus 多 index 复杂生态。**Filterable HNSW + ACORN 集成**是 Qdrant 独特的 filter-aware HNSW 路径。

## 架构图

[per sources/docs/qdrant/overview/what-is-qdrant.md + distributed_deployment.md + manage-data/storage.md]

```
┌──────────────────────────────────────────────────┐
│  Clients (Python / TS / Rust / Java / Go / .NET) │
├──────────────────────────────────────────────────┤
│  HTTP REST (6333) / gRPC (6334)                  │
├──────────────────────────────────────────────────┤
│  Cluster Layer (Raft Consensus)                  │
│  ├─ Collection metadata + topology (Raft log)    │
│  ├─ Point updates (eventual consistency)         │
│  └─ Peer-to-peer (6335)                          │
├──────────────────────────────────────────────────┤
│  Per-Node                                        │
│  ├─ Collection(s)                                │
│  │   └─ Shard(s) [consistent hash / user-defined]│
│  │       └─ Segment(s) [appendable + non-app.]   │
│  │           ├─ Vector Storage                   │
│  │           │   ├─ in-memory (RAM)              │
│  │           │   └─ memmap (mmap, on disk)       │
│  │           ├─ Payload Storage                  │
│  │           │   ├─ in-memory (default)          │
│  │           │   └─ on-disk (RocksDB / Gridstore)│
│  │           ├─ Vector Index (HNSW only)         │
│  │           │   └─ Filterable HNSW (extra edges)│
│  │           ├─ Payload Index (per field)        │
│  │           │   ├─ keyword/int/float/bool/geo/dt│
│  │           │   ├─ text (tokenizer + stemmer)   │
│  │           │   ├─ uuid                         │
│  │           │   ├─ tenant index (multitenancy)  │
│  │           │   └─ principal index (time/key)   │
│  │           └─ Sparse Vector Index (IDF)        │
│  └─ WAL (Write-Ahead-Log) for durability         │
├──────────────────────────────────────────────────┤
│  Storage (local disk / network mounts)           │
└──────────────────────────────────────────────────┘
```

## 数据模型

[per sources/docs/qdrant/overview/what-is-qdrant.md + manage-data/collections.md, points.md, payload.md]

### Core abstractions

| Entity | 描述 |
|---|---|
| **Point** | (id, vector(s), payload) tuple——Qdrant 的基本存储单位 |
| **Vector** | 高维 float32 / uint8 / 稀疏向量 |
| **Payload** | JSON object 附加在 point；可被 indexed (8 种类型) |
| **Collection** | 命名的 points 集合；vector dimensionality + distance metric 全集合统一 |
| **Named Vectors** | 单 point 可有多个 vector（不同 dim/metric） |
| **Sparse Vectors** | First-class citizen since v1.7.0 |
| **Segment** | Collection 内独立 store；vector index + payload index per segment |
| **Shard** | 分布式部署单位；shard_number 创建时定，consistent hashing 分配 |

### Distance Metrics

`Cosine` / `Dot` / `Euclid` / `Manhattan`（Sparse 仅 `Dot`）

> Cosine 实现为 normalized dot product（vectors auto-normalize on upload）

## 关键设计决策

### 1. Filterable HNSW with payload-aware extra edges（v1.0+）

[per sources/docs/qdrant/manage-data/indexing.md "Filterable HNSW Index"]

**问题**：HNSW + post-filter 在 middle selectivity 失败：
- High-selectivity (loose) filter → HNSW 直接 work
- Low-selectivity (strict) filter → payload index + rescore
- **Middle**：full scan 太多 / HNSW graph 在 strict filter 下崩溃

**Qdrant solution**：**HNSW graph + payload-aware extra edges**——为每个 indexed payload field 添加额外 edge，让 search 在 graph 内 jump 到符合 filter 的 candidate。

```
HNSW topology = base layer 0 graph + ... + top layers
+ Extra edges: 对每 indexed payload field F:
    在 graph 中找 same-F-value 的 close points 加 edge
    → search 时 filter 同 F-value 不需 exhaustive
```

**关键约束**（per indexing.md aside）：
> Extra edges for HNSW graph can **only be generated after payload index creation**.
> It's highly recommended to create all payload indices **immediately after collection creation, before ingesting data**.

→ Build order: collection → payload indexes → ingest → HNSW build with extra edges。

**与已 ingest filter-aware ANN 方法对比**：

| | [FilteredVamana](../concepts/filtered-vamana.md) | [ACORN-γ](../concepts/acorn.md) | **Qdrant Filterable HNSW** |
|---|---|---|---|
| Base graph | Vamana (α-controlled) | HNSW (γ-density factor) | **HNSW (m + ef_construct)** |
| Filter 集成 | Label baked-in to RobustPrune | **Predicate-agnostic + γ candidate edges + 2-hop search** | **Per-payload-index extra edges** |
| Filter cardinality | LCPS (≤1000) | HCPS (10^8+) unbounded | **bounded per payload-indexed field** |
| Production status | Microsoft 广告 A/B +35-49% revenue | Stanford 学术 only (until Qdrant v1.16.0) | **Qdrant production** |

### 2. ACORN integration as Filterable HNSW fallback（v1.16.0+）—— ACORN 首次 production

[per sources/docs/qdrant/manage-data/indexing.md "The ACORN Search Algorithm"]

> *Available as of v1.16.0*  ...the additional edges built for Qdrant's filterable HNSW may not be sufficient. These extra edges are added for each payload index **separately, but not for every possible combination** of payload indices. As a result, a combination of two or more strict filters might still lead to disconnected graph components. ...In such cases, use the ACORN Search Algorithm.

**Qdrant 集成 ACORN 的方式**：
- Filterable HNSW (extra edges) 是默认 path
- ACORN 算法（2-hop neighbor expansion）作 **fallback**：当 (a) 多个 strict filter 组合 OR (b) 大量 soft-deleted points 时启用
- Trade-off: 提升 recall，损失 search performance

→ **wiki 内 [ACORN](../concepts/acorn.md) frontier 关闭**——Patel et al. 2024 SIGMOD 论文 Stanford 学术工作于 2026 年通过 Qdrant 进入 production。

### 3. Tenant Index + Principal Index（v1.11.0+）—— payload-aware storage layout

[per sources/docs/qdrant/manage-data/indexing.md "Tenant Index" / "Principal Index"]

- **Tenant Index** (keyword / uuid only): 把 same-tenant data 在 disk 上 colocate，减少 cross-tenant reads
- **Principal Index** (integer / float / datetime): 主 filter 字段（如 timestamp）优化 disk layout——time-range 查询 friendly

→ 这是 wiki 内**首个 payload-aware data layout 优化**——与 [Starling Block Shuffling](../concepts/block-shuffling.md) 的 graph-layout 优化是**正交的另一维度**（payload 字段 vs vertex 邻居）。

### 4. Quantization 多变体覆盖（v1.1.0 - v1.15.0）

[per sources/docs/qdrant/manage-data/quantization.md]

| Method | Version | Compression | Note |
|---|---|---|---|
| **Scalar Quantization** (SQ) | v1.1.0 | **4×** | float32 → uint8, SIMD-friendly |
| **Binary Quantization** (BQ) | v1.5.0 | **32×** + 40× speedup | float32 → 1 bit; Hamming via dot product |
| **2-bit Quantization** | v1.15.0 | 16× | -1 / 0 / 1 三 buckets，小维度更准 |
| **1.5-bit Quantization** | v1.15.0 | 24× | merged 2-bit buckets，中间精度 |
| **Asymmetric Quantization** | v1.15.0 | 32× store + scalar query | Binary stored + Scalar query—磁盘 I/O bottleneck 友好 |
| **Product Quantization** (PQ) | v1.2.0 | 16-32× | k-means 256 centroids per chunk |

**与 wiki quantization landscape 关系**：
- Scalar/PQ 是 wiki 已覆盖
- **Binary + 1.5/2-bit + Asymmetric** 是 Qdrant 在 binary path 的细化（wiki 内之前仅 indirect 提及）
- 未集成 [RaBitQ](../concepts/rabitq.md)（unbiased + sharp error bound）——理论上 RaBitQ + Qdrant Filterable HNSW 是 logical work

### 5. Collection Aliases for model migration

[per sources/docs/qdrant/manage-data/collections.md "Collection aliases"]

> ...sometimes necessary to switch different versions of vectors seamlessly. **For example, when upgrading to a new version of the neural network.** ...all changes of aliases happen **atomically**, no concurrent requests will be affected during the switch.

**Workflow**:
```
1. Old collection `prod_v1` (旧 model embeddings) serve queries via alias `prod`
2. Build new collection `prod_v2` (新 model embeddings) in background
3. Migrate data via Qdrant Migration tool: `qdrant-migration` docker image
4. Atomic alias swap: `prod` → `prod_v2`, old `prod_v1` deleted/archived
```

→ **wiki 内 18 个 ingest 中首次明确 production migration tooling**——但仍是工程层（atomic switch + migration tool），不解决跨 model embedding mapping 的 algorithm 问题。详见 [topics/in-place-vs-out-of-place-updates.md](../topics/in-place-vs-out-of-place-updates.md) embedding-update-handling 部分。

### 6. Raft consensus + sharding（v0.8.0+）

[per sources/docs/qdrant/distributed_deployment.md]

- **Raft** for collection metadata + topology consensus
- **Point updates 不走 Raft**——eventual consistency，可 `wait` 参数等待
- **Sharding 二选一**：(a) automatic via consistent hashing, (b) user-defined (v1.7.0+)
- **Replication factor** + **write_consistency_factor** 配置
- **Resharding** v1.13.0 仅 Cloud（OSS 需创建新 collection）

### 7. Storage 选项 spectrum

[per sources/docs/qdrant/manage-data/storage.md]

| Storage option | RAM cost | Speed |
|---|---|---|
| Vector: in-memory | high | **highest** |
| Vector: memmap (mmap) | low | depends on page cache |
| Payload: InMemory (default) | high | high |
| Payload: OnDisk (RocksDB) | low | slow (推荐配 payload index 减少 disk reads) |
| HNSW index: in-memory (default) | medium | high |
| HNSW index: on-disk (`hnsw_config.on_disk=true`) | low | medium |
| Payload index: in-memory (default) | low-medium | high |
| Payload index: on-disk (v1.11.0+) | low | medium |

→ Qdrant **提供完整 spectrum**——从全 in-memory 到全 on-disk，按 collection / per-vector / per-field 配置。

## Scale 边界

[per sources/docs/qdrant/distributed_deployment.md + capacity-planning.md]

| 配置 | 实证 |
|---|---|
| Single-node | 推荐 dev / 非 production |
| 2-node cluster | 平衡 cost + 部分 resilience |
| **3-node cluster + replication** | Production 推荐 |
| 12 shards | 支持 expand 到 1/2/3/6/12 nodes 不需 re-shard |
| Qdrant Cloud 商用规模 | 不公开具体数字 |
| Cluster resharding | v1.13.0+ Cloud only |

## 与 wiki 已有系统的对比

### 与 [Milvus](./milvus.md)（同代开源 vector DBMS）

| | Milvus | **Qdrant** |
|---|---|---|
| 实现 | Go + C++ kernel | **Rust 单 binary** |
| 团队 | Zilliz | Qdrant Solutions GmbH |
| Index 多样性 | 多（HNSW/IVF*/DISKANN/CAGRA/SPARSE...） | **HNSW only** + Sparse |
| Filter 策略 | 5 partition-based strategies | **Filterable HNSW + ACORN** |
| Quantization | PQ / SQ8 / ScaNN | **Scalar / Binary / 1.5/2-bit / Asym / PQ** |
| Cloud-native | Disaggregated (Manu) | **Cloud + Hybrid + Private + Edge** |
| Streaming | LSM segment + Manu growing index | WAL + segment optimizer |
| Update model | LSM merge | aliases + migration tool |
| Multi-vector query | Vector fusion + iterative merging | Named vectors + hybrid query |
| 商业实证 | 300+ enterprise | Qdrant Cloud 客户 |

→ Milvus 走"多 index_type + 复杂 cloud-native disaggregated"路径；Qdrant 走"单 index + Rust 简洁"路径。**架构哲学正交**。

### 与 [Pinecone](./pinecone.md)（闭源 SaaS）

| | Pinecone | **Qdrant** |
|---|---|---|
| License | 闭源 | Apache-2.0 OSS |
| 部署 | SaaS only | OSS + Cloud + Hybrid + Private + Edge |
| Index 选择 | adaptive 黑盒 | HNSW 明示 |
| User control | 配置受限 | **完全自主调参** |
| 透明度 | 文档但底层不开源 | **算法 + 实现完全可读** |
| Filter | metadata 黑盒 | **Filterable HNSW + ACORN** |
| Migration | API | **Collection aliases atomic switch** |

→ Pinecone 是 SaaS 黑盒便利路径；Qdrant 是 OSS 完全透明路径。各占工业场景。

### 与 [Faiss](./faiss.md)（library）

Faiss 是 library（非 DBMS），Qdrant 是完整 DBMS。Qdrant 可视为"Rust 实现的 Faiss equivalent + DBMS 上层（API + WAL + sharding + multitenancy + cloud SKU）"。**不直接竞争**——Faiss 作为算法库可被 Qdrant-style DBMS 包装。

### 与 [VBASE](./vbase.md)（iterator + RM）

VBASE engine layer Iterator + [RM](../concepts/relaxed-monotonicity.md) 攻击 K' problem；Qdrant 仍是 TopK 接口（无 iterator）。理论上 Qdrant 可加 RM iterator 接口（HNSW 满足 RM），但 OSS 当前未做。

### 与 [FreshDiskANN](./freshdiskann.md)（graph streaming）

FreshDiskANN 是 Vamana base streaming；Qdrant 是 HNSW base + segment-based optimizer + WAL。Qdrant 不需要 α-RNG（α-augmented RobustPrune）——HNSW + segment LSM merge 模式天然 streaming-friendly（但 recall 演化未在 Qdrant docs 详细量化）。

### 与 [CAGRA](./cagra.md)（GPU graph）

CAGRA 是 NVIDIA RAPIDS RAFT 的 GPU-native graph；Qdrant 是 CPU + Rust 路径。**未直接集成 CAGRA**——但理论上 Qdrant 可 + NVIDIA RAFT bindings (类似 Milvus GPU_CAGRA)。Open work。

## 生产案例

[per qdrant.tech site + benchmarks]

- **Qdrant Cloud SaaS**: Stripe, AT&T, Disney, Mozilla, X, Bayer, AWS Hybrid 等
- **Edge deployment**: Qdrant Edge runs on commodity hardware/devices for on-device search
- **GitHub stars (as of 2026)**: ~25k+ stars，活跃社区
- **Customer references**: 见 [qdrant.tech/customers/](https://qdrant.tech/customers/)

## Open Questions

- **Qdrant + [RaBitQ](../concepts/rabitq.md)**: Qdrant 当前 BQ 1.5/2-bit + Asymmetric 是 binary path 细化；RaBitQ unbiased + sharp bound 理论上提供更优 → 集成 logical 但 docs 未提
- **Qdrant + [CAGRA](./cagra.md) GPU**: Milvus 已有 GPU_CAGRA via RAFT；Qdrant 当前 Rust + CPU only，GPU 路径未明
- **Qdrant + iterator + [RM](../concepts/relaxed-monotonicity.md) (VBASE-style)**: HNSW 满足 RM；iterator API 可加。当前 TopK only
- **Qdrant streaming vs [FreshVamana](../concepts/freshvamana.md) / [SPFresh](./spfresh.md)**: Qdrant 用 WAL + segment optimizer + collection aliases，没有 α-RNG 这种 explicit streaming graph property。streaming 性能曲线 wiki 未深入量化
- **Qdrant 千亿 scale 实证**: docs 不公开具体千亿规模数字；Qdrant Cloud production 数据未在 OSS docs
- **ACORN production performance Qdrant vs Patel 2024 paper benchmarks**: Qdrant v1.16.0 集成 ACORN——production workload 与论文 25M LAION 实测的 gap 未量化
- **Filterable HNSW vs FilteredVamana head-to-head**: 两种 filter-aware build 在同 dataset 的对比 wiki 内 zero coverage
- **Collection Aliases + cross-model embedding mapping**: aliases 是工程便利（atomic switch），不解决 vector space 跨 model 不一致问题——algorithm 层仍 zero coverage
- **Qdrant Edge deployment**: 资源约束下（commodity device）的 ANN 实证细节 docs 仅 high-level
- **HNSW + Sparse vector hybrid 性能**: Qdrant first-class sparse vector，与 Milvus SPARSE_INVERTED_INDEX 同代——head-to-head 未实证
- **Qdrant vs [Vespa](./vespa.md) HNSW + filter 哲学对比**：两者都是 ACORN production case（Qdrant v1.16.0 fallback + Vespa "Acorn-1" mode），但 filter 一等公民程度不同。Qdrant "do HNSW exceptionally well"——HNSW + payload-aware extra edges + ACORN fallback；Vespa "HNSW 是众多算子之一"——HNSW 嵌入 4-phase ranking pipeline，与 BM25 / weakAnd / tensor compute 同列为 retrieval operator。**filter integration 路径不同**：Qdrant 是 graph build-time aware（Filterable HNSW + tenant index）；Vespa 是 query-time orchestration（pre-filter / post-filter / Acorn-1 三种 mode 由 YQL planner 选）。Qdrant 索引视野窄但深（HNSW only + 多 quantization）；Vespa 索引视野广（HNSW + Streaming + BM25 + tensor framework）但每个不一定最深。详见 [systems/vespa.md "ACORN 与 filter integration"](./vespa.md)。

Cited by: 待 query 引用
