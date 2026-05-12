---
title: Weaviate（Go 实现 AI-native primary vector DBMS）
type: system
sources: [weaviate-docs, vespa-docs, turbopuffer-docs, formal-2021-splade-v2]
related: [milvus.md, qdrant.md, pinecone.md, vespa.md, turbopuffer.md, faiss.md, vbase.md, analyticdb-v.md, pase.md, spfresh.md, freshdiskann.md, cagra.md, ../concepts/hnsw.md, ../concepts/acorn.md, ../concepts/filtered-vamana.md, ../concepts/rabitq.md, ../concepts/product-quantization.md, ../concepts/relaxed-monotonicity.md, ../concepts/freshvamana.md, ../concepts/splade-sparse-retrieval.md, ../topics/attribute-filtering.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md, ../topics/in-place-vs-out-of-place-updates.md, ../topics/topk-vs-iterator-model.md, ../topics/sparse-dense-hybrid-retrieval.md]
created: 2026-05-11
updated: 2026-05-12 (SPLADE as sparse-side hybrid next-gen)
---

# Weaviate

**TL;DR**: Weaviate B.V. 2019 开源的 **Go 实现 vector DBMS**（BSD-3-Clause），定位为 **"AI-native primary database"**——把 objects + vectors + inverted indexes 集成在同一系统（不仅是 vector store，是完整 DB）。**核心独特性**：(1) **HNSW + RQ8 quantization default**——RQ (Rotational Quantization) 与 [RaBitQ](../concepts/rabitq.md) 同源（random rotation + quantize）但选 8-bit per dim 而非 1-bit；HFresh preview 是 HNSW base 的 streaming/freshness 工程化；(2) **ACORN 集成**——wiki 内**第二个 ACORN production case**（继 [Qdrant](./qdrant.md) v1.16.0 之后），证明 [patel-2024-acorn] Stanford 学术工作已成 OSS vector DBMS default 之一；(3) **First-class hybrid search** (BM25 + vector)——`col.query.hybrid()` 单一 API，BM25 用 **BlockMaxWAND** 算法；(4) **完整 inverted index 套件**：BlockMaxWAND BM25 + **Roaring bitmaps** for set filters + **bit-sliced range bitmaps** for range filters + LSM store；(5) **Vertical stack**：Core DB + Cloud + **Query Agent** (turnkey RAG) + **Engram** (agent memory preview) + Plugins/Cookbooks——是 wiki 内首个**显式集成 agent stack 上层产品的 vector DBMS vendor**；(6) **gRPC primary** (50051) + HTTP (8080) for metadata——GraphQL deprecated。**Multi-tenancy + RBAC** first-class（collection / tenant-level permissions, default on v1.30+）。**Built-in embeddings** `text2vec-weaviate` 无需第三方 API key。[per sources/docs/weaviate/llms.txt]

## 与 wiki 现有系统的定位差异

[per qdrant-docs / pinecone-docs / wang-2021-milvus Table 1 + sources/docs/weaviate/llms.txt]

| | [Milvus](./milvus.md) | [Pinecone](./pinecone.md) | [Qdrant](./qdrant.md) | **Weaviate** |
|---|---|---|---|---|
| 实现语言 | Go + C++ kernel | 闭源 | Rust | **Go** |
| License | Apache-2.0 | 闭源 SaaS | Apache-2.0 | **BSD-3-Clause** |
| 形态 | OSS + Zilliz Cloud | 闭源 SaaS only | OSS + Cloud/Hybrid/Private/Edge | **OSS + Cloud + BYOC + Dedicated** |
| 定位 | "Vector data management system" | "Vector database SaaS" | "Vector search engine" (Rust) | **"AI-native primary database"** |
| Vector + non-vector | Vector + scalar payload | Vector + metadata | Vector + payload (JSON) | **Vector + objects + full LSM/inverted index 套件** |
| Index 多样性 | 多 (HNSW/IVF*/DISKANN/CAGRA/SPARSE) | 黑盒 adaptive | **HNSW only** + Sparse | **HNSW only** (+ HFresh preview) + Sparse |
| Hybrid search (BM25+vector) | SPARSE_INVERTED_INDEX + reranker | Sparse-dense hybrid | Sparse vectors first-class | **`col.query.hybrid()` API + BlockMaxWAND BM25** |
| Default quantization | 不强默认 | 黑盒 | 显式可选 (Scalar/Binary/PQ) | **RQ8 default** (与 RaBitQ 同源不同粒度) |
| Streaming freshness | LSM segment + Manu growing | Pinecone slabs | LSM segment + WAL | **HFresh preview** (HNSW + freshness) |
| Filter-aware ANNS | 5 partition strategies | metadata 黑盒 | Filterable HNSW + ACORN | **ACORN + query planning + positive/negative correlation** |
| Multi-tenancy | partition + RBAC | namespace | Tenant Index + Principal Index | **Collection-level multi-tenancy config + RBAC (per-collection/tenant)** |
| Model migration | n/a | n/a | **Collection Aliases atomic** | (未在 llms.txt 显式提及) |
| Agentic stack | n/a | Inference API | n/a | **Query Agent + Engram + Plugins** (vendor-integrated) |
| Built-in embeddings | n/a | Pinecone Inference | n/a | **text2vec-weaviate (recommended)** |
| API 协议 | gRPC + REST | REST + Python SDK | gRPC + REST + Python SDK | **gRPC primary (50051) + HTTP (8080) for metadata** |
| Sharding | segment + LSM | slab adaptive | Raft + consistent hashing | **Sharding + replica movement, Cloud auto-scaling** |
| 商业体量 | 300+ enterprise | 闭源 SaaS 客户 | OSS + Cloud (Stripe/AT&T/Disney/Mozilla) | OSS + Cloud (开放生态) |

**核心论点**：Weaviate 在 wiki 内**填补 AI-native primary DB 定位**——比 Milvus / Qdrant 更明确"不只是 vector store"，比 Pinecone 更"DB-like"。**Vertical agent stack** (Query Agent + Engram) 是其差异化武器。

## 架构图

[per sources/docs/weaviate/llms.txt §Architecture/Scaling]

```
┌─────────────────────────────────────────────────────┐
│  Clients (Python / TypeScript / Go / Java / C#)     │
├─────────────────────────────────────────────────────┤
│  gRPC (50051) [data ops]  +  HTTP (8080) [metadata] │
├─────────────────────────────────────────────────────┤
│  Weaviate Server (Go)                               │
│  ┌──────────────────────────────────────────────┐   │
│  │  Schema / Auto-schema                        │   │
│  │  Multi-tenancy + RBAC (collection + tenant)  │   │
│  ├──────────────────────────────────────────────┤   │
│  │  Query Planner                               │   │
│  │  ├─ Hybrid search (BM25 + vector fusion)     │   │
│  │  ├─ Filter planning (ACORN + +/-correlation) │   │
│  │  └─ Aggregations                             │   │
│  ├──────────────────────────────────────────────┤   │
│  │  Indexes per Collection (per Shard)          │   │
│  │  ├─ Vector: HNSW + RQ8 (default)             │   │
│  │  │   or HFresh (preview, streaming-friendly) │   │
│  │  │   + ACORN for strict filter combinations  │   │
│  │  ├─ BM25: BlockMaxWAND inverted index        │   │
│  │  ├─ Filter: Roaring bitmaps (set/equality)   │   │
│  │  └─ Range: Bit-sliced bitmaps (opt-in)       │   │
│  ├──────────────────────────────────────────────┤   │
│  │  Storage: LSM store for objects + indexes   │   │
│  │  + WAL for durability                        │   │
│  └──────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────┤
│  Sharding + Replication (horizontal scale)          │
│  ├─ Replica movement                                │
│  ├─ Auto-scaling (Cloud)                            │
│  └─ Separation of control plane / data plane (Cloud)│
├─────────────────────────────────────────────────────┤
│  Storage (local disk / Cloud-managed)               │
└─────────────────────────────────────────────────────┘

Optional Agent Stack (Cloud):
  ┌─────────────────────────────────────────────────┐
  │  Query Agent (turnkey RAG: PDF ingest + chunk)  │
  │  Engram (agent memory, preview)                  │
  │  Agent Plugins / Cookbooks                       │
  └─────────────────────────────────────────────────┘
```

## 数据模型

[per sources/docs/weaviate/llms.txt §Quickstart / Misconceptions]

### Core abstractions

| Entity | 描述 |
|---|---|
| **Collection** | 命名 set of objects + vector(s) + indexes (former name: Class) |
| **Object** | Data row containing properties + (optionally) named vectors |
| **Property** | Object field with data type (TEXT/NUMBER/etc.) + index config |
| **Named Vectors** | 多 vectors per object（如 title + body 各自向量化） |
| **Sparse vectors** | First-class via SPARSE_INVERTED_INDEX equivalent |
| **Tenant** | Multi-tenant isolation primitive (collection-level config) |
| **Shard** | Horizontal scale unit (auto-sharded by Cloud, manual on OSS) |

### Property index defaults

[per sources/docs/weaviate/llms.txt §Filtering]

```python
# Auto-schema default (no explicit property config):
#   index_filterable = True   (roaring bitmap for equality/set filters)
#   index_searchable = True   (BM25 map for keyword/hybrid search)
#   index_range_filters = False  (opt-in for </> queries)
#   tokenization = "word"     (alphanumeric, lowercased)
```

→ 默认 collection 即可执行 hybrid query；range query 需要显式 enable。

## 关键设计决策

### 1. HNSW + RQ8 default quantization

[per sources/docs/weaviate/llms.txt §Architecture]

> Vector index: **HNSW with RQ8 quantization by default** — good recall/speed tradeoff for most workloads.

- **RQ (Rotational Quantization)**: random orthogonal rotation + scalar quantize 8-bit per dim → 4× memory compression
- **概念上与 [RaBitQ](../concepts/rabitq.md) 同源**——都是 "random rotation makes data distribution uniform → quantize uniformly"——但粒度不同：RaBitQ 1-bit per dim (32×), Weaviate RQ 8-bit per dim (4×)
- **RQ1** (1-bit/dim) also available for cost-sensitive workloads (per llms.txt §Best Practices)
- **HFresh preview**: better freshness/streaming guarantees on HNSW base——这是 wiki 内首个 HNSW base streaming graph 工程（之前 [FreshVamana](../concepts/freshvamana.md) 是 Vamana base + α=1.2）。Weaviate docs 未公开 HFresh 算法细节

### 2. ACORN production deployment

[per sources/docs/weaviate/llms.txt §Key Features]

> Advanced filtering & sorting (eq, neq, range, sort by) **with ACORN**, query planning, positive & negative correlation optimization

- **wiki 内第二个 ACORN production case**（继 Qdrant v1.16.0）
- "Positive & negative correlation optimization" 是 Weaviate 额外的 query planner enhancement——基于 payload 字段间相关性估计 selectivity
- ACORN 启用条件 / fallback logic 未在 llms.txt 详述

→ [concepts/acorn.md "Open Questions"] 中 ACORN production frontier 进一步关闭——两个独立 OSS vector DBMS 都集成 ACORN，证明已成主流 default。

### 3. First-class hybrid search

[per sources/docs/weaviate/llms.txt §Quickstart + sources/docs/weaviate/hybrid-search.md]

**API design philosophy**: `col.query.hybrid(query="...", limit=3)` 单一 API，无需 tuning。

**关键算法选择**：
- BM25: **BlockMaxWAND** (高性能 top-k 文本搜索)
- 融合：vector similarity + BM25 score 加权（default α 通过 query 自动调整）
- 默认 hybrid 是 Weaviate **推荐 default query type** ("use hybrid search for highest result quality")

**与 wiki 内 hybrid-search-equivalent 系统对比**：

| 系统 | Hybrid 实现 | API |
|---|---|---|
| Weaviate | BlockMaxWAND BM25 + HNSW vector + 自动融合 | `col.query.hybrid()` 单 API |
| Milvus | SPARSE_INVERTED_INDEX + dense + reranker | 多 vector field 配合 multi-vector query |
| Qdrant | Sparse vectors first-class (IDF modifier) + dense | hybrid query API |
| Pinecone | Sparse-dense hybrid | hybrid query |

→ Weaviate API 是最简化的（单函数调用），其他系统需要更多 schema/query 配置。

### 4. Inverted index 完整套件

[per sources/docs/weaviate/llms.txt §Architecture]

| Index 类型 | 实现 | 用途 | Default |
|---|---|---|---|
| Objects + properties | **LSM store** | 主存储 | ✓ |
| BM25 full-text | **BlockMaxWAND** | 文本 + hybrid search | ✓ (per property) |
| Set / equality filter | **Roaring bitmaps** | filter by category/tag/tenant | ✓ |
| Range filter | **Bit-sliced range bitmaps** | filter by price/timestamp | **Opt-in** (`index_range_filters=True`) |
| Vector | **HNSW + RQ8** (or HFresh preview) | vector similarity | ✓ |

> **Without explicit opt-in, range queries fall back to a full scan.** ——重要 gotcha for range-heavy workloads。

→ 这是**完整的传统数据库 inverted index 套件**集成在 vector DBMS 内——Weaviate "primary DB not just vector store" 的核心技术 evidence。

### 5. Vertical agent stack

[per sources/docs/weaviate/llms.txt §The Weaviate Stack + agentic-ai.md + rag.md]

| 组件 | 用途 | 状态 |
|---|---|---|
| **Core DB** (Go, OSS) | 生产 vector DBMS | GA |
| **Weaviate Cloud** (DBaaS) | Zero-ops scaling | GA |
| **Query Agent** (Cloud only) | Turnkey RAG: PDF ingest + auto-chunking + retrieval | GA |
| **Engram** (Cloud only, preview) | Agent memory: auto-extract/inject/update memories from conversations | **Preview** |
| **Agent Plugins / Cookbooks** | e2e application examples | varies |

→ **wiki 内首个明确集成 agent stack 上层产品的 vector DBMS vendor**。Pinecone 有 Inference API 但不及 Weaviate 上层产品深度。

### 6. Built-in embeddings (`text2vec-weaviate`)

> Use `weaviate-embeddings` with the default model — optimized for most users: cost-effective, accurate, no 3rd-party API keys required.

→ **Recommended default**——不需要 OpenAI / Cohere / Voyage 等第三方 embedding API key。与 [Pinecone Inference](./pinecone.md) 同代但更显式 default。20+ 第三方 integrations 仍可选。

### 7. gRPC primary + HTTP for metadata

[per sources/docs/weaviate/llms.txt §Misconceptions]

> Weaviate uses **gRPC (default port 50051) for data operations** and **HTTP (8080) for metadata/REST**. All official clients use gRPC under the hood.

→ 与 wiki 已 ingest [Milvus](./milvus.md) (gRPC) / [Qdrant](./qdrant.md) (gRPC + REST) 一致——modern vector DBMS 协议标配。**GraphQL deprecated**——历史包袱已清。

### 8. Multi-tenancy + RBAC first-class

[per sources/docs/weaviate/llms.txt §Multi-tenancy / RBAC]

- **Multi-tenancy**: Collection-level config (`multi_tenancy_config=enabled`); 每 tenant 独立 shard storage
- **RBAC** (v1.29+, default on v1.30+): per-collection + per-tenant permissions; user management API + OIDC

```python
client.collections.create(
    "Docs",
    multi_tenancy_config=Configure.multi_tenancy(enabled=True),
)
col.tenants.create([Tenant(name="tenantA"), Tenant(name="tenantB")])
tenant_col = col.with_tenant("tenantA")
```

→ 比 Qdrant Tenant Index 更高粒度（collection-level vs payload-field-level）。

### 9. Auto-schema by default (Cloud)

[per sources/docs/weaviate/llms.txt §Misconceptions / Schema]

- Cloud: auto-schema default
- Self-hosted: 需 env var enable
- 用户**可以零 schema 配置 ingest data**——schema 仅在需要 fine-tune specific feature (range filters, tokenization) 时才显式定义

## Scale 边界

[per sources/docs/weaviate/llms.txt §Architecture/Scaling + cost-performance-optimization.md]

| 配置 | 实证 |
|---|---|
| Cloud free trial | 2 周 demo |
| Cloud (managed, auto-scaling) | "scales to any production workload" (具体上限不公开) |
| Self-hosted Docker/k8s | 用户自管 sharding + replication |
| Dedicated tier | "predictable throughput, isolation, compliance" |
| BYOC | Bring your own cloud |

> **Scale-out**: Horizontal scalability via sharding & replica movement. Fully managed on Weaviate Cloud.
> **Zero-downtime**: All maintenance ops on Weaviate Cloud rely on replication.

具体千亿规模实证未在 llms.txt 公开。

## 与 wiki 已有系统的对比

### 与 [Qdrant](./qdrant.md)（同代 OSS HNSW DBMS）

| | Qdrant (Rust) | **Weaviate (Go)** |
|---|---|---|
| 哲学 | "Vector search engine + simplicity" | **"AI-native primary database"** |
| Vector + 其他数据 | Payload (JSON) + payload index | **Full objects + LSM + inverted index 套件** |
| Hybrid search | Sparse vectors first-class | **`col.query.hybrid()` 单 API + BlockMaxWAND** |
| Quantization 默认 | 显式可选 (Scalar/Binary/PQ) | **RQ8 default** |
| ACORN 集成 | v1.16.0 fallback | **常驻 + query planner correlation 优化** |
| Streaming | LSM segment + WAL | **HFresh preview (HNSW base streaming)** |
| Multi-tenancy | Tenant Index + user-defined sharding | **Collection-level multi_tenancy + RBAC** |
| Vendor stack | OSS + Cloud + Hybrid + Private + Edge | OSS + Cloud + BYOC + Dedicated + **Query Agent + Engram** |
| Model migration | Collection Aliases atomic switch | (llms.txt 未显式提及) |

→ **Qdrant 偏 "Rust 性能 + 简洁"；Weaviate 偏 "AI-native primary DB + agent stack"**——两条并列 OSS 路径。

### 与 [Milvus](./milvus.md)（同代 Go OSS DBMS）

| | Milvus | **Weaviate** |
|---|---|---|
| 实现 | Go + C++ kernel | **纯 Go** |
| 架构 | 4 层 disaggregated (Manu) | **更简洁** (单 binary + sharding) |
| Index 多样性 | 多 (HNSW/IVF*/DISKANN/CAGRA/SPARSE) | **HNSW only** + HFresh |
| 商业体量 | 300+ enterprise | 不公开具体数字 |
| Agent stack | n/a | **Query Agent + Engram** |
| Vertical productization | Zilliz Cloud | **Cloud + Query Agent + Engram + Plugins** |

→ Milvus 更"Tool box for all workloads"；Weaviate 更"AI-native primary DB"。**生态定位不同**。

### 与 [Pinecone](./pinecone.md)（闭源 SaaS）

| | Pinecone | **Weaviate** |
|---|---|---|
| License | 闭源 | BSD-3-Clause OSS |
| 部署 | SaaS only | OSS + Cloud + BYOC + Dedicated |
| 透明度 | docs 但底层不开源 | **算法 + 实现完全可读** |
| Index 选择 | 黑盒 adaptive | **HNSW + RQ8 显式** |
| Hybrid search | sparse-dense | **`col.query.hybrid()`** |
| 商业 Agent stack | Pinecone Inference | **Query Agent + Engram (更上层)** |
| 商业体量 | 闭源 SaaS 客户 | OSS + Cloud 开放生态 |

→ Pinecone 是 SaaS 黑盒便利；Weaviate 是 OSS + Cloud + 显式 agent stack。

### 与 [Faiss](./faiss.md) / [DiskANN](./diskann.md) 等 library

Faiss 是 library，Weaviate 是完整 DBMS——不直接竞争。DiskANN 同样不是 DBMS。Weaviate 可视为"Go 实现的 OSS vector DBMS 集成多种 indexing + 上层 agent stack"——Faiss/DiskANN 算法可被 Weaviate-style DBMS 包装。

### 与 [VBASE](./vbase.md) (PostgreSQL iterator + RM)

VBASE engine layer Iterator + RM 攻击 K' problem；Weaviate 仍 TopK 接口 (no explicit iterator)。理论上 Weaviate 可加 RM iterator (HNSW 满足 RM)，但当前未做。

### 与 [PASE](./pase.md) (PostgreSQL ANN extension)

PASE 是 PG 内嵌；Weaviate 是 standalone DB——架构起点不同。**8b OLTP-extended-vector** vs **8a vector-first DBMS** (vector-first 定位)。

### 与 [CAGRA](./cagra.md) (GPU graph)

CAGRA NVIDIA GPU；Weaviate Go CPU——硬件 path 不同。Weaviate 未集成 CAGRA。

### 与 [FreshDiskANN](./freshdiskann.md) / [SPFresh](./spfresh.md)

Weaviate HFresh preview 是 HNSW base streaming——与 FreshDiskANN (Vamana base) / SPFresh (SPANN cluster base) 形成**第三条 streaming 路径** (HNSW base)。但 HFresh 算法细节未在 docs 公开。

## 生产案例

[per Weaviate Cloud customer list + docs]

- **Open source**：~13k+ GitHub stars，活跃社区
- **Cloud customers**：开放生态广泛但具体客户不公开 specifics
- **Vertical Stack adoption**：Query Agent + Engram preview——early adopter case studies 在 docs blog

## Open Questions

- **HFresh 算法细节**：preview 状态，docs 未公开内部机制。理论上是 HNSW + α-augmented RobustPrune 类似 [FreshVamana](../concepts/freshvamana.md) 思路，但 Weaviate 未声明 algorithm
- **RQ vs RaBitQ 实测对比**：两者同源（random rotation + quantize）但粒度不同（8-bit vs 1-bit）；同硬件 head-to-head wiki 内 zero coverage
- **ACORN production performance Weaviate vs Qdrant vs Patel 2024 paper**：两个 OSS 都集成但实测 vs 学术 25M LAION 的 gap 未量化
- **"Positive & negative correlation optimization"**：Weaviate query planner 用 payload 字段相关性估计 selectivity——具体算法 docs 未公开；可能是简单 covariance 估计也可能更复杂
- **Hybrid search 融合公式**：BM25 + vector score 的具体融合 α、normalization 策略 docs 未深入
- **Weaviate 千亿规模实证**：docs 不公开具体数字
- **Weaviate vs Qdrant production benchmark**：双方都不公开 head-to-head
- **HFresh vs FreshDiskANN graph-path streaming**：Weaviate HFresh (HNSW base) vs FreshDiskANN FreshVamana (Vamana base)——两个 graph-path streaming 实现的算法/性能比较 wiki 内 zero coverage
- **Engram 算法**：agent memory "auto-extract/inject/update" 内部细节 docs preview 未深入；与传统 LLM context window management 的差异未量化
- **Model migration in Weaviate**：llms.txt 未显式提及 Collection Alias-style migration tool（Qdrant 已明确）；Weaviate 是否有等价机制？未确认
- **gRPC vs Milvus gRPC 设计差异**：都用 gRPC + protocol 但 service 接口 / streaming 用法 wiki 未对比
- **AI-native primary DB 的工业实证**：Weaviate llms.txt 反复强调"primary DB not just vector store"——实际客户中"replace Postgres + Pinecone with Weaviate" 案例 docs 不公开
- **Weaviate vs [Vespa](./vespa.md) "primary DB" 哲学对比**：两者都自称 vector + 多 modality primary DB，但 lineage 与权衡不同。Weaviate "AI-native primary DB built around vectors"——2019 Go 起步，vector-first 但补 inverted index + agent stack (Query Agent / Engram)；Vespa "web search engine (Yahoo! 2003) + vector retrofit"——20 年 inverted index + ranking pipeline + tensor framework 沉淀，2017 OSS，后加 HNSW。**Ranking 成熟度差距明显**：Vespa 4-phase ranking（retrieval → first-phase → second-phase → global-phase）+ first-class tensor framework (ONNX/XGBoost/LightGBM native)——专为 LTR / 复杂 ranking pipeline 设计；Weaviate 用 BM25 + vector + 简单 fusion (α-blend)，ranking pipeline 更轻。**filter / hybrid search 路径不同**：Weaviate ACORN + "positive & negative correlation optimization"（payload-aware planner）；Vespa pre-filter / post-filter / Acorn-1 三 mode + YQL planner。Weaviate 优势：agent stack 集成（Query Agent / Engram memory）；Vespa 优势：ranking pipeline 复杂度上限。详见 [systems/vespa.md "4-phase ranking"](./vespa.md)。
- **Weaviate vs [Turbopuffer](./turbopuffer.md) "AI-native" 同代不同 storage 哲学**：两者都自称 AI-native vector DBMS，但 storage 主从关系完全不同。Weaviate "vector + objects + inverted index 套件 + agent stack"——NVMe local primary，传统 LSM；Turbopuffer "object storage as primary"——S3 是 source of truth, compute fully stateless。**ANN 算法路径分歧**：Weaviate HNSW (graph-based) + RQ8 quantization default + HFresh streaming preview；Turbopuffer SPFresh (centroid-based) only。**两者都强调 multi-tenancy 但模型不同**：Weaviate "collection-level multi-tenancy + RBAC"——complex collection-tenant hierarchy；Turbopuffer **namespace as architectural primitive**（100M+ S3 prefix）——更 aggressive 的 namespace-as-tenant。**Agent / app stack 集成**：Weaviate Query Agent + Engram first-class；Turbopuffer 明示"focused on first-stage retrieval"，2nd stage rerank / agent logic 推到 application 层（`search.py` / `search.ts`）——**stack 边界相反**。**OSS vs closed**：Weaviate BSD-3-Clause OSS；Turbopuffer commercial-only with BYOC。详见 [systems/turbopuffer.md "对比"](./turbopuffer.md)。
- **Weaviate first-class hybrid + [SPLADE](../concepts/splade-sparse-retrieval.md) migration path (NEW 2026-05-12 ingest)**: Weaviate `collection.query.hybrid(query=..., alpha=...)` API 是 wiki 内**最早 first-class hybrid retrieval API** (BlockMaxWAND BM25 + dense vector α-blend, default α=0.5). 当前 sparse path 是经典 BlockMaxWAND BM25; **SPLADE (neural sparse) 是 next-gen 替代 candidate**——客户可改 BM25 输入 from raw text → SPLADE-encoded sparse vector, 同 `hybrid()` API. Weaviate 也支持 user-provided sparse vector (via Inference module 或 manual upload), 这是 SPLADE production deployment 自然路径. **优势** vs Vespa rank-profile: Weaviate single-API 更 ergonomic (alpha 单参数 control), vs Vespa expression 需 ranking-profile 编写. **劣势**: 仅 1D α-blend (vs Vespa 任意 expression + 4-phase ranking). Weaviate Reranker module 提供 application-side cross-encoder rerank (e.g., Cohere Rerank-v3, mxbai-rerank). **Open**: (a) Weaviate 客户 BM25 → SPLADE 实际 migration cost / quality gain 不公开; (b) BlockMaxWAND BM25 索引能否复用 to 服务 SPLADE-encoded sparse vector? Weaviate 内部 sparse index 实现细节 docs 不深入. 详见 [topics/sparse-dense-hybrid-retrieval.md](../topics/sparse-dense-hybrid-retrieval.md).

Cited by: 待 query 引用
