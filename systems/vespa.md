---
title: Vespa（C++/Java search + recommendation engine with first-class tensor）
type: system
sources: [vespa-docs, radford-2021-clip]
related: [milvus.md, qdrant.md, pinecone.md, weaviate.md, faiss.md, spann.md, diskann.md, vbase.md, analyticdb-v.md, pase.md, freshdiskann.md, ../concepts/hnsw.md, ../concepts/acorn.md, ../concepts/filtered-vamana.md, ../concepts/product-quantization.md, ../concepts/rabitq.md, ../concepts/relaxed-monotonicity.md, ../concepts/clip.md, ../topics/attribute-filtering.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md, ../topics/in-place-vs-out-of-place-updates.md, ../topics/topk-vs-iterator-model.md, ../topics/multi-vector-queries.md, ../topics/multimodal-embedding-retrieval.md]
created: 2026-05-11
updated: 2026-05-11 (CLIP as multimodal embedding peer)
---

# Vespa

**TL;DR**: Yahoo! 2003 内部开发的 **search + recommendation + personalization engine**, 2017 开源 (Apache-2.0), 现由 Vespa.ai 维护。C++ + Java 实现。**与 wiki 内其他 vector DBMS lineage 不同**：Vespa 来自 web search engine 而非 vector store 演化——所以 vector search 仅是 Vespa 多功能之一，与 BM25 / structured filter / ranking / tensor framework 同级。**4 大独特点**：(1) **SPANN production deployment**——wiki 内继 Microsoft Bing 之后**第二个 SPANN production**（OSS 路径，Microsoft SPACEV 10M-100M sample）；(2) **"Acorn-1" mode**——wiki 内**第三个 ACORN production**（继 Qdrant + Weaviate, **ACORN frontier 完全闭合**）；(3) **Streaming Search**——no-index brute-force within per-user subset, **45 bytes per document in memory**, billions of documents per node——与所有其他 ANN 系统**哲学相反**（针对 personal data scenarios）；(4) **4-phase ranking pipeline** (retrieval → first-phase → second-phase → global-phase + reranking searcher) + **first-class tensor framework** (dense/sparse/mixed) + ONNX/XGBoost/LightGBM/TensorFlow native integration——是 wiki 内最复杂的 ML-integrated ranking 系统。**架构**：Stateless container clusters (Java + ONNX inference) + Stateful content clusters (auto-distributed HNSW/SPANN/streaming) + atomic Application Package deployment。**Vespa Cloud** + Vespa Operator K8s + self-hosted Linux + Docker。**Application Package** (services.xml + schemas/*.sd) 是 atomic deployment unit。

## 与 wiki 现有系统的定位差异

[per qdrant-docs / weaviate-docs / milvus-docs / wang-2021-milvus Table 1 + sources/docs/vespa/llms.txt overview]

| | [Milvus](./milvus.md) | [Pinecone](./pinecone.md) | [Qdrant](./qdrant.md) | [Weaviate](./weaviate.md) | **Vespa** |
|---|---|---|---|---|---|
| 实现语言 | Go + C++ kernel | 闭源 | Rust | Go | **C++ + Java** |
| 起源 | Vector DBMS (Zilliz 2019+) | Vector DBMS SaaS (Pinecone 2019+) | Vector search engine (Qdrant 2021+) | Vector DBMS (Weaviate 2019+) | **Web search engine (Yahoo! 2003+)** |
| Lineage | Vector-first 演化 | Vector-first 演化 | Vector-first 演化 | Vector-first 演化 | **Web search engine → vector 集成** |
| License | Apache-2.0 | 闭源 | Apache-2.0 | BSD-3-Clause | **Apache-2.0** |
| Open sourced | 2019 | 闭源 | 2021 | 2019 | **2017 (after 14 yrs 内部)** |
| 形态 | OSS + Zilliz Cloud | SaaS only | OSS + Cloud/Hybrid/Private/Edge | OSS + Cloud + BYOC + Dedicated | **OSS + Vespa Cloud + K8s Operator** |
| 定位 | "Vector data management system" | "Vector database SaaS" | "Vector search engine" | "AI-native primary database" | **"Search + Recommendation + Personalization engine"** |
| Vector + 其他数据 | Vector + scalar payload | Vector + metadata | Vector + payload (JSON) | Vector + objects + LSM | **Vector + documents + tensor fields + 完整 search/ranking stack** |
| Index 多样性 | 多 (HNSW/IVF*/DISKANN/CAGRA/SPARSE) | 黑盒 | HNSW only | HNSW only + HFresh preview | **HNSW + SPANN + Streaming + 多 quantization** |
| Filter-aware | 5 strategies | metadata | Filterable HNSW + ACORN | ACORN + correlation | **Pre-filter / post-filter / Acorn-1 (filter before distance)** |
| Ranking 深度 | 简单 phased | 黑盒 | 简单 hybrid | hybrid + reranker | **4-phase ranking + ONNX/XGBoost/LightGBM/TensorFlow native + tensor expressions + cross-encoder global-phase + custom Java reranker** |
| BM25 实现 | basic | n/a | sparse vectors | BlockMaxWAND | **WAND + weakAnd + sparse top-k + nativeRank multi-feature** |
| Tensor framework | n/a | n/a | n/a | n/a | **First-class typed tensors (dense/sparse/mixed) + ONNX integration** |
| Streaming search (no-index) | n/a | n/a | n/a | n/a | **Unique: 45 bytes/doc + billions/node + per-user subset** |
| Multi-vector per doc | named vectors | n/a | named vectors | named vectors | **Multi-vector indexing (map of vectors), `tensor<float>(i{},x[512])`** |
| Application package | n/a | n/a | API config | API config | **Atomic deployable package (services.xml + schemas/*.sd)** |
| Custom Java components | n/a | n/a | n/a | n/a | **Stateless container hosts Searchers + Document Processors** |
| 商业体量 | 300+ enterprise | 闭源 SaaS | OSS + Cloud | OSS + Cloud | **Yahoo! Mail / Yahoo! News / Spotify / various enterprises** |

**核心论点**：Vespa 是 wiki 内**唯一来自 web search engine lineage 的系统**——其他都是 vector store 演化。所以 Vespa 的 BM25 / structured query / ranking / tensor framework / personalization features 都是 first-class 设计，不是"vector store 上加"。**Vector search 是 Vespa 多功能之一**。

## 架构图

[per sources/docs/vespa/llms.txt overview + llms-full.txt §Phased Ranking + §Streaming Search]

```
┌──────────────────────────────────────────────────┐
│  Clients (HTTP REST + Java SDK + Python SDK)     │
├──────────────────────────────────────────────────┤
│  Stateless Container Clusters (Java)             │
│  ┌──────────────────────────────────────────┐    │
│  │  YQL Parser + Query API (/search/)       │    │
│  │  Document API (/document/v1/)            │    │
│  │  Custom Java Searchers + Document Procs  │    │
│  │  Global-phase ranking + cross-encoder    │    │
│  │  ONNX accelerated inference (CPU/GPU)    │    │
│  │  Federation + query rewriting + scatter  │    │
│  └──────────────────────────────────────────┘    │
├──────────────────────────────────────────────────┤
│         ↓ gRPC scatter-gather ↓                  │
├──────────────────────────────────────────────────┤
│  Stateful Content Clusters (C++)                 │
│  ┌──────────────────────────────────────────┐    │
│  │  Per-shard:                              │    │
│  │  ├─ Vector index                         │    │
│  │  │   ├─ HNSW (primary, mutable)          │    │
│  │  │   ├─ SPANN (billion-scale hybrid)     │    │
│  │  │   └─ Streaming (no-index, per-user)   │    │
│  │  ├─ Inverted index (BM25 + WAND)         │    │
│  │  ├─ Attribute store (in-memory + paged)  │    │
│  │  ├─ Document store (LSM)                 │    │
│  │  ├─ First-phase ranking (all hits)       │    │
│  │  ├─ Second-phase ranking (top 100/node)  │    │
│  │  └─ Match features                       │    │
│  └──────────────────────────────────────────┘    │
├──────────────────────────────────────────────────┤
│  Application Package (atomic deployable)         │
│  ├─ services.xml (cluster topology)              │
│  ├─ schemas/*.sd (documents + rank profiles)     │
│  ├─ models/ (ONNX / XGBoost / LightGBM / TF)     │
│  └─ components/ (custom Java)                    │
├──────────────────────────────────────────────────┤
│  ConfigServer + Vespa Cloud / Vespa Operator     │
└──────────────────────────────────────────────────┘
```

## 关键设计决策

### 1. **Stateless Container + Stateful Content** separation

[per sources/docs/vespa/llms.txt overview]

- **Container** (Java): 处理 query parsing, federation, custom Searchers/DocProcs, global-phase ranking with ONNX
- **Content** (C++): 数据存储 + vector index + inverted index + first/second-phase ranking
- **Auto data distribution + redundancy** in content cluster
- 比 Milvus 4-layer disaggregated 更**清晰二分**——container = compute layer, content = storage + per-shard compute

### 2. **Application Package** atomic deployment

[per sources/docs/vespa/llms.txt §Application Package]

```
my-app/
├── services.xml          ← cluster topology + resource allocation
├── schemas/
│   ├── doc.sd            ← document type + rank profile (Vespa native DSL)
│   └── ...
├── components/           ← custom Java Searchers + DocProcs
└── models/               ← ONNX / XGBoost / LightGBM files
```

→ **Atomic deployable**——配置 + 代码 + ML model 一起部署，保证 code-config consistency。这是 web search engine 历史的产物（生产 deployment 严肃）。

### 3. **HNSW with comprehensive filtering**

[per sources/docs/vespa/llms-full.txt §Approximate Nn Hnsw]

- **Vespa 实现 modified HNSW** (基于 [malkov-2016-hnsw](../concepts/hnsw.md) 论文)
- **Pre-filtering** (default) + **Post-filtering** + **"Acorn-1"** (filtering before distance calculation)
- `approximate-threshold` + `post-filter-threshold` + `filter-first-threshold` 三参数控制 strategy
- **Multi-vector indexing**: `tensor<float>(i{},x[512])` 一个 field 多 vectors（map of vectors）
- **Multi-field**: 同 schema 多 HNSW 字段（不同 model / source）
- **Multithreaded indexing** (HNSW build 多线程)
- **Mutable HNSW graph** (no segmented/partitioned, 一 graph per node per field)
- **Multi-value types**: `double / float / bfloat16 / int8 / single-bit`

**HNSW 配置 schema 示例**:
```
field text_embedding type tensor<float>(x[384]) {
  indexing: summary | attribute | index
  attribute {
    distance-metric: prenormalized-angular
  }
  index {
    hnsw {
      max-links-per-node: 24
      neighbors-to-explore-at-insert: 200
    }
  }
}
```

### 4. **SPANN production deployment**（wiki 内 SPANN 第二个 production）

[per sources/docs/vespa/llms-full.txt §Billion Scale Vector Search]

> The SPANN (Space Partitioned ANN) approach... SPANN searches for the k closest centroid vectors of the query vector in the in-memory ANN search data structure. Then, it reads the k associated posting lists for the retrieved centroids...

- 实现基于 [chen-2021-spann](./spann.md) SPANN 论文
- **HNSW (in-memory centroid graph)** + **posting lists (data on disk)** hybrid
- Sample app 使用 **Microsoft SPACEV 10M-100M** production-style dataset
- Vespa Cloud 与 self-hosted 都支持

→ wiki 内 SPANN production:
1. **Microsoft Bing** (chen-2021 论文实证)
2. **Vespa** (OSS implementation, this ingest)

详见 [systems/spann.md "production deployments"](./spann.md)。

### 5. **"Acorn-1" mode for filtering before distance calculation**

[per sources/docs/vespa/llms-full.txt §Approximate Nn Hnsw]

> ANN searches in Vespa support both pre-and post-filtering, beam exploration, and **filtering before distance calculation ("Acorn 1")**.

- Vespa 实现 [patel-2024-acorn](../concepts/acorn.md) 的 ACORN-1 算法
- Filter 在 distance 计算之前应用——比传统 pre-filter 更激进
- 与 Qdrant fallback / Weaviate "correlation optimization" 形成 3rd ACORN production

→ wiki 内 ACORN production:
1. **Qdrant v1.16.0** (fallback mode)
2. **Weaviate** (default + correlation optimization)
3. **Vespa** (Acorn-1 = filter before distance)

→ **ACORN frontier 完全闭合**：3 个独立 OSS DBMS 都实现 = 工业 default。

### 6. **Streaming Search**: no-index brute-force within per-user subset

[per sources/docs/vespa/llms-full.txt §Streaming Search]

> For use cases where data is split into many small subsets where each query just searches one (or a few) of these subsets, the canonical example being _personal indexes_ where a user only searches their own data...Vespa provides _streaming search_—**no indexes required**. ...**45 bytes per document** in memory, meaning that streaming mode lets you store **billions of documents on each node**.

**Streaming search 与 ANN 系统哲学相反**：

| | HNSW / SPANN / 典型 ANN | **Streaming Search (Vespa)** |
|---|---|---|
| 设计前提 | Cross-corpus search | **Single user search 自己数据** |
| Index | 必建（HNSW graph / IVF posting） | **不建** |
| Memory per doc | hundreds of bytes (graph + vector) | **45 bytes** (raw doc summary stored) |
| Docs per node | million-scale | **billions** |
| Search complexity | sublinear | **linear within group** |
| ANN | approximate | **always exact** |
| Persistence | mostly in-memory | **raw doc on disk + attributes on disk** |
| 典型 use case | Web search / RAG / cross-corpus retrieval | **Email / personal notes / chat history / per-tenant SaaS** |

**Implementation details**：
- `document mode="streaming"` in services.xml
- Document IDs 含 `group` value (typically userid)
- Query 指定 `streaming.groupname` 参数
- 自动 sharding 大 group 跨多 content nodes 并行
- **HNSW indexes are NOT supported in streaming search**——所有 NN search are exact

→ **Vespa 是 wiki 内唯一明确支持 per-user search 优化**的系统。Pinecone namespace / Qdrant Tenant Index / Weaviate multi-tenancy 都是基于 partition 但仍建 ANN index；Streaming Search 完全不建 index——是另一种 trade-off。

### 7. **4-phase ranking pipeline**

[per sources/docs/vespa/llms-full.txt §Phased Ranking]

```
Retrieval (sub-linear top-k)
  ├─ nearestNeighbor (HNSW / SPANN)
  ├─ weakAnd / WAND (sparse top-k)
  └─ YQL filter combination
    ↓
First-phase (per content node, ALL retrieved hits)
  ├─ first-phase expression (BM25, freshness, etc.)
  └─ rank-score-drop-limit (filter low-score hits)
    ↓
Second-phase (per content node, top 100/node by first-phase)
  ├─ second-phase expression (XGBoost / LightGBM / function)
  └─ total-rerank-count (global cap)
    ↓
Global-phase (stateless container, top N globally after scatter-gather)
  ├─ ONNX model inference (cross-encoder)
  ├─ Cross-hit normalization (reciprocal rank fusion)
  └─ rerank-count
    ↓
Reranking Searcher (custom Java)
```

**4 个 ranking 层** —— wiki 内最复杂 ML-integrated ranking 系统。

**ML integration**:
- **First/second-phase** (content nodes): XGBoost / LightGBM / 任意 ranking expression (basic tensors + features)
- **Global-phase** (stateless container): ONNX models (cross-encoders) + GPU inference acceleration
- **Match features** + **rank features** 跨 phase 数据传递

**ML model file** 直接 deployed in Application Package。

### 8. **First-class Tensor Framework**

[per sources/docs/vespa/llms-full.txt §Ranking + Tensor Examples + Tensor User Guide]

Vespa 的 tensor 是 ranking expressions 的 **native primitive**——不像其他 vector DBMS "vector is special data type"：

```
field embedding type tensor<float>(x[768]) { ... }
field categories type tensor<float>(category{}) { ... }    # sparse
field user_pref type tensor<float>(category{},x[768]) { ... }   # mixed
```

**Cell types**: `double / float / bfloat16 / int8 / single-bit`

**Tensor operations** in rank expression:
```
rank-profile my_rank {
    first-phase {
        expression: sum( query(q_vec) * attribute(doc_vec), x )
    }
}
```

→ **Vespa = ranked search engine that natively understands tensors as first-class data**。Vector 仅是 1-dim tensor。

### 9. **Multi-value vector types + Binary quantization**

[per sources/docs/vespa/llms-full.txt §Binarizing Vectors]

```
Cell types (per dimension):
  double:       8 bytes
  float:        4 bytes
  bfloat16:     2 bytes (50% savings, negligible accuracy loss)
  int8:         1 byte  (often pre-quantized embeddings)
  single-bit:   0.125 byte (Hamming-style; with re-ranking)
```

- Binary quantization + re-ranking with full precision on disk (paged attributes)
- 与 Qdrant Binary 1.5/2-bit + Asymmetric / Weaviate RQ8/RQ1 同代 production binary quantization

### 10. **Auto-scaling content cluster**

[per sources/docs/vespa/llms.txt overview + llms-full.txt §Sizing Search]

> Content clusters can be grown or shrunk on the fly without service interruptions, and data is automatically redistributed to maintain a balanced load.

- Real-time CRUD + low-latency indexing
- 节点 add/remove 时**数据自动 rebalance**
- 类似 Qdrant resharding (Cloud only) 但 Vespa 是 OSS 一致 supported

## Scale 边界

[per sources/docs/vespa/llms-full.txt § Billion Scale Image Search + Billion Scale Vector Search + Sizing Search + Streaming Search]

| 配置 | 实证 |
|---|---|
| Single-node | dev/test |
| Multi-node OSS | 推荐 production |
| **Billion-scale HNSW + filtering** | Vespa Cloud + AWS instances |
| **SPANN billion-scale** | Microsoft SPACEV 10M-100M production sample |
| **LAION-5B 5.85B image-text pairs** | Vespa Cloud + CLIP ViT-L/14 + PCA + HNSW-IF hybrid |
| **Streaming search** | **billions of documents per node** (45 bytes/doc) |
| Vespa Cloud production | Yahoo! Mail / Yahoo! News / Spotify / etc. (具体规模不公开) |

→ Vespa 在**多种 scale + 多种 workload**下 production——比 wiki 已 ingest OSS vector DBMS 更广 deployment matrix。

## 与 wiki 已有系统的对比

### 与 [Milvus](./milvus.md)（OSS DBMS, 类似 scale）

Vespa 是 **search engine lineage**，Milvus 是 **vector DBMS lineage**。两者**核心差异**：

| | Milvus | **Vespa** |
|---|---|---|
| 起源 | Zilliz 2019 vector DBMS | Yahoo! 2003 web search engine |
| Lineage | vector-first | search-first + vector 集成 |
| Index 选项 | HNSW/IVF*/DISKANN/CAGRA/SPARSE | **HNSW + SPANN + Streaming** |
| Ranking | basic phased | **4-phase + ONNX/XGBoost/LightGBM/TF native** |
| Tensor framework | n/a | **first-class** |
| BM25 + structured | basic | **WAND + weakAnd + nativeRank + complex YQL** |
| Streaming search (no-index) | n/a | **unique** |
| 团队 | Zilliz (vector startup) | Vespa.ai (Yahoo! veterans, ~20 yrs IR experience) |

→ Milvus 优势在"多 index_type + cloud-native"；Vespa 优势在"成熟 search engine + ranking depth + tensor framework + multiple scale modes"。

### 与 [Qdrant](./qdrant.md)（OSS HNSW-focused）

Qdrant 简洁专一 (Rust + HNSW only)；Vespa 复杂全面 (C++ + Java + HNSW + SPANN + Streaming + 4-phase ranking + tensor framework)。**起始复杂度** Vespa 远高于 Qdrant；**集成深度** Vespa 远超 Qdrant。

### 与 [Weaviate](./weaviate.md)（OSS AI-native primary DB）

- Weaviate "AI-native primary DB" + agent stack；Vespa "search engine 集成 vector + tensor"
- Weaviate first-class hybrid (col.query.hybrid); Vespa more flexible via YQL + phased ranking
- Weaviate Query Agent + Engram (RAG turnkey)；Vespa custom Java + ONNX (DIY but more powerful)

### 与 [Pinecone](./pinecone.md)（闭源 SaaS）

- Pinecone simple SaaS;Vespa OSS + Vespa Cloud (managed by experts)
- Pinecone adaptive index black-box; Vespa user-control everything via schema

### 与 [Faiss](./faiss.md) / [DiskANN](./diskann.md) (library/algorithm)

Vespa 是 DBMS;Faiss / DiskANN 是 algorithm libraries. Vespa 自家实现 HNSW (不 wrap Faiss); SPANN (基于 Microsoft 论文 reimpl)。

### 与 [VBASE](./vbase.md)

VBASE iterator + RM (PostgreSQL extension)；Vespa 4-phase ranking (different paradigm)。两者都解决"K' 预测问题"但 approaches 不同——VBASE engine layer iterator vs Vespa multi-phase explicit ranking。

### 与 [SPFresh](./spfresh.md) / [FreshDiskANN](./freshdiskann.md)

Vespa HNSW 是 mutable graph + real-time CRUD（per llms-full.txt 明示）——但 docs **不公开 streaming 性能曲线**。理论上类似 hnswlib add-only 但 docs 说 "Remove vectors in real time"——可能 implementation 处理 deletion 通过 tombstone + 定期 merge，类似 Milvus LSM 而非 FreshVamana α-augmented 思路。具体细节 docs 不深入。

### 与 [CAGRA](./cagra.md)（GPU graph）

Vespa **GPU support for ONNX inference in global-phase** （ranking layer），但 ANN vector index 本身仍 CPU。理论上 Vespa stateless container 可以加 NVIDIA RAFT bindings 但 docs 未明示。

## 生产案例

[per Vespa public references + llms.txt overview]

- **Yahoo! Mail** / **Yahoo! News** / **Yahoo!** 各种搜索/推荐——Vespa 内部 user (起源)
- **Spotify** (search / discovery features 部分使用)
- **Various enterprises** 通过 Vespa Cloud
- **LAION-5B 5.85B image-text** sample app on Vespa Cloud
- GitHub stars: ~5k+ (vespa-engine/vespa)

## Open Questions

- **Vespa HNSW streaming insert/delete 性能曲线**：docs 提"real-time CRUD"但具体 50 cycles × 5% change recall stability 未公开（与 [FreshVamana](../concepts/freshvamana.md) 对比）。可能用 tombstone + merge 类似 LSM，未深入
- **Vespa SPANN vs Microsoft Bing SPANN 实测对比**：双方都不公开具体 latency/recall 数字
- **Streaming Search billion-doc/node 实证**：docs 给"billions per node"理论容量，但 production 实证 case studies 不公开
- **4-phase ranking 在 千亿 scale 的 P99 latency**：docs 不公开
- **Vespa + [RaBitQ](../concepts/rabitq.md)**: Vespa 当前 binary quantization 不集成 RaBitQ unbiased + sharp bound；logical 但未实证
- **Vespa GPU + CAGRA**: Stateless container 已 GPU for ONNX; vector index 自身 CAGRA-style GPU 路径 docs 未明示
- **Vespa Streaming Search + RAG with Engram-style agent memory**: Vespa 没 explicit agent stack 上层产品 (vs Weaviate Query Agent + Engram)；docs 推荐用户用 Vespa stateless container 自己实现
- **Vespa vs Milvus / Qdrant / Weaviate head-to-head benchmark**: 完全空白；Vespa.ai 偶尔在 blog 提及 ann-benchmarks.com 结果但不公开 production comparison
- **Application Package 在多 region production**: services.xml topology + deployment.xml；docs 提及但生产实例不深入
- **Custom Java Searchers debugging / observability** in production: docs 多 reference 但实战 case studies 缺
- **Vespa Cloud vs self-hosted Vespa Operator** scale ceiling: Cloud 不公开规模上限；Operator-based K8s self-host 仍 docs reference
- **Vespa + [CLIP](../concepts/clip.md) multimodal native pattern (NEW 2026-05-11 ingest)**: Vespa 是 wiki 内**最适合 CLIP-style multimodal retrieval production** 的 vector DBMS——(1) tensor framework first-class (`tensor<float>(x[768])` 直接存 ViT-L/14 embedding, ONNX/native CLIP encoder 可 inline 跑); (2) 多 tensor field per doc 支持 image_embedding + caption_embedding 共存; (3) 4-phase ranking pipeline 把 CLIP first-stage retrieval + reranker second-stage 一体化, vs peer OSS DBMS 把 rerank 推到 application 层; (4) position dimension + tensor + BM25 多模态 + spatial 同 schema first-class——目前 wiki 唯一支持 multimodal + spatial 同时 native 的 vector DBMS. **Open**: (a) Vespa-inline CLIP encoder vs application-level CLIP encoder latency / cost 实测对比? docs 不深入; (b) Vespa multimodal 大规模 production case (>100M images) 公开数据? unknown; (c) CLIP retrieval + ranking-profile (4-phase) 的具体 production query template? docs 简单. 详见 [topics/multimodal-embedding-retrieval.md](../topics/multimodal-embedding-retrieval.md) 与 [concepts/clip.md](../concepts/clip.md).

Cited by: [queries/giga-scale-sharding.md](../queries/giga-scale-sharding.md)
