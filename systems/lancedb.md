---
title: LanceDB（multimodal lakehouse for AI, OSS embedded + Enterprise）
type: system
sources: [lancedb-docs, lancedb-docs-2026-05, lance-geo-blog-2026-02]
related: [chroma.md, pgvector.md, milvus.md, faiss.md, turbopuffer.md, pinecone.md, ../concepts/hnsw.md, ../concepts/product-quantization.md, ../concepts/rabitq.md, ../concepts/cagra-graph.md, cagra.md, ../topics/gpu-vs-cpu-ann.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md, ../topics/multimodal-embedding-retrieval.md, ../topics/sparse-dense-hybrid-retrieval.md, ../topics/ann-benchmarking-methodology.md, ../benchmarks/vectordbbench.md]
created: 2026-05-12
updated: 2026-05-29 (补 §H 原生空间索引: Lance R-Tree + GeoArrow + GeoDataFusion, 2026-02 新增 — 纠正此前"无 spatial"判断)
---

# LanceDB

**TL;DR**: LanceDB 是 **"multimodal lakehouse for AI"** 哲学的 vector DBMS, 由 Chang She + Lei Xu 2022 创立 (Series A 2024). OSS Apache-2.0 embedded library (Python / TypeScript / Java / Rust) + LanceDB Enterprise (distributed managed multimodal lakehouse). **对 wiki 内 vector DBs 的核心独特性**: (1) **Lance columnar format = 独特存储 axis**——所有 wiki 内其他 vector DBMS 都用 vendor-specific storage (Milvus segment / Qdrant single binary / Chroma SQLite / pgvector Postgres heap / Vespa tensor / Pinecone slab / Turbopuffer object storage); LanceDB 用 **OSS Lance columnar format** (类似 Parquet 但 ML-optimized + vector index integration)——**wiki 内首个 vendor 以 OSS columnar format 作 primary storage substrate**, 与 Iceberg / Delta / Hudi 这类 lakehouse format philosophy 并列; (2) **Multimodal lakehouse 哲学**——单一 table 同时存 vector + metadata + 原始 multimodal data (text / image / video / point cloud) + 版本控制 + zero-copy ops——其他 vector DBs 普遍 vector + metadata only, raw data 推到 separate object storage; (3) **OSS embedded library + Enterprise 双模式** (类似 Chroma)——OSS dev / Enterprise petabyte-scale managed, 但**架构 unique**: 基于 Lance format 不切换 codebase (vs Chroma OSS Core HNSW → Cloud SPANN 不同 codebase); (4) **GPU index building（claim 待核实）**——2026-05-12 首轮 ingest 据 docs 摘要记为"first-class GPU index build",但 **2026-05 深化抓取的 indexing/quantization 文档完全未提 GPU build**——该 claim 现降级为待核实（见 Open Questions），不再作为 LanceDB 的确定差异化卖点; (5) **SQL query support + ML framework integration**——同 platform 内支持 LangChain / LlamaIndex / DuckDB / Pandas / Polars 直接 query, **超越纯 vector DB 边界进入 data engineering / feature engineering 领域**; (6) **Zero-copy versioning**——schema evolution + add new columns 不需 copy existing data, "table-level git-like" version control; (7) **Polyglot SDKs**: Python / TypeScript / Java / Rust 4 语言一等公民, Rust core + 多语言 binding. **Position**: 与 Chroma 同 "OSS embedded + Cloud" 双模式, 但走 **lakehouse-first** 而非 RAG-dev-first 路线. [per sources/docs/lancedb/]

## 与 wiki 内其他 system 的定位差异

| | Milvus | Qdrant | Weaviate | Vespa | Pinecone | Turbopuffer | pgvector | Chroma | **LanceDB** |
|---|---|---|---|---|---|---|---|---|---|
| Storage substrate | segment (custom) | single Rust binary | Go binary | tensor + custom | slab (闭源) | object storage primary | Postgres heap | SQLite (OSS) / object storage (Cloud) | **Lance columnar format (OSS)** |
| Storage format 是否 OSS standard | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | Postgres native | partial | **✓ Lance OSS format** (类比 Iceberg/Delta/Hudi) |
| Embedded library 模式 | ✗ (DBMS only) | ✗ (DBMS) | ✗ (DBMS) | ✗ (engine) | ✗ (SaaS only) | ✗ (SaaS) | extension | **✓ (OSS Core)** | **✓ (OSS) — primary mode** |
| Multimodal raw data + vector 同 table | partial (多 vector field) | multiple vectors per point | named vectors | tensor + content | 不公开 | namespace-per-asset | sparsevec / vector type only | text + metadata + vector | **✓ vector + metadata + raw multimodal (image / video / point cloud)** |
| GPU index build | ✓ via CAGRA (Milvus integration) | ✗ | ✗ | ✗ | 不公开 | ✗ | ✗ | ✗ | **? 待核实**（2026-05 indexing/quantization docs 未提及） |
| SQL query support | partial (limited) | API | API | YQL | API | API | **full SQL native** | API | **✓ SQL + Python/JS/Java/Rust polyglot** |
| Version control | snapshot-based | snapshot | snapshot | application package | namespace | namespace | row-level via SQL transactions | **CoW fork (Cloud)** | **zero-copy table versioning + schema evolution** |
| ML framework integration | application | application | application | ONNX inline | application | application | application | LangChain / LlamaIndex (vendor-bundled) | **LangChain / LlamaIndex / DuckDB / Pandas / Polars native first-class** |

**核心论点**：LanceDB **不与上述 vendor 在 "pure vector DB" 维度竞争**——LanceDB 走 **"lakehouse-first"**: 把 vector retrieval 作为 multimodal lakehouse 的一个 capability, 与 OLAP query + feature engineering + ML training 一体. 是 wiki 内**唯一明示 "data lakehouse + vector retrieval" 集成哲学** 的 vendor.

## 架构图

```
              LanceDB OSS embedded library
              ┌──────────────────────────────────────┐
              │  Python / TypeScript / Java / Rust   │
              │  SDK                                  │
              └──────────────┬───────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────────────┐
              │  LanceDB Rust core engine             │
              │                                       │
              │  Query (vector + SQL + FTS)           │
              │  Index (IVF*/HNSW family; GPU build?) │
              └──────────────┬───────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────────────┐
              │  Lance columnar format                │
              │  (OSS, Apache-2.0)                    │
              │                                       │
              │  • Parquet-like columnar              │
              │  • Optimized for ML / random access   │
              │  • Vector index storage native        │
              │  • Schema evolution + versioning      │
              │  • Multimodal (image / video / point  │
              │    cloud + metadata + embedding 同表) │
              └──────────────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────────────┐
              │  Object storage / local disk          │
              │  (S3 / GCS / Azure / NVMe)            │
              └──────────────────────────────────────┘

              LanceDB Enterprise
              ┌──────────────────────────────────────┐
              │  Distributed managed multimodal       │
              │  lakehouse                            │
              │                                       │
              │  Same Lance format core + 分布式      │
              │  query engine + managed indexes        │
              │  + petabyte-scale ops                 │
              └──────────────────────────────────────┘
```

[per sources/docs/lancedb/README.md + Lance format README]

## 数据流 / 控制流

### 1. Create table + insert multimodal data

```python
import lancedb
import pyarrow as pa

db = lancedb.connect("./data/sample.lance")

table = db.create_table("products", data=[
    {
        "id": "prod-1",
        "name": "Italian leather wallet",
        "image": b"<binary image bytes>",  # raw multimodal
        "embedding": [0.1, 0.2, ...],       # 768-d vector
        "price": 199.99,
        "category": "wallet"
    }
])
```

**核心特点**:
- Lance format 同表存储 vector + metadata + raw multimodal (image binary 直接 column)
- 不需要 separate object storage for raw images (其他 vector DBs 通常需要)

### 2. Vector + SQL hybrid query

```python
# Pure vector query
result = table.search([0.12, 0.22, ...]).limit(10).to_pandas()

# Vector + SQL filter
result = table.search([0.12, 0.22, ...]) \
    .where("category = 'wallet' AND price < 500") \
    .limit(10) \
    .to_pandas()

# SQL-only query (no vector)
df = table.to_pandas(filter="category = 'wallet'")
```

→ LanceDB 通过 SQL filter + vector search 一体——比 pgvector SQL native 但 standalone (不需 Postgres).

### 3. Versioning + zero-copy schema evolution

```python
# Add new column without copying existing data
table.add_columns({"new_embedding_v2": "embeddings(...)"})  # zero-copy

# Query historical version
table_at_version_5 = table.checkout(version=5)
```

→ **Table-level versioning**——类似 git for data, 不复制 existing rows. 与 Chroma collection fork (CoW) 哲学相近但 axis 不同 (LanceDB column-level + Chroma collection-level).

### 4. ML framework integration native

```python
# Direct to Pandas
df = table.to_pandas()

# Direct to DuckDB
import duckdb
duckdb.sql("SELECT * FROM table WHERE price < 500")

# LangChain integration
from langchain.vectorstores import LanceDB
vectorstore = LanceDB(table=table)
```

→ **wiki 内唯一 vendor 把 ML framework integration 作 first-class data path** (其他都通过 application-bundled connector).

## 关键设计决策

### 1. Lance columnar format as primary storage substrate

[per sources/docs/lancedb/lance-format-README.md]

vs 其他 vendor proprietary storage:
- Milvus: segment-based proprietary format
- Qdrant: Rust binary internal storage
- Weaviate: Go binary internal LSM
- Vespa: tensor-based proprietary
- Pinecone: slab (黑盒)
- Turbopuffer: object storage SPFresh internal
- Chroma: SQLite (OSS) / object storage (Cloud)
- pgvector: Postgres heap (general-purpose)
- **LanceDB: Lance format (OSS Apache-2.0 columnar format)**

**Lance format key properties**:
- 类似 Parquet 列存但 ML-optimized (fast random access for slicing / sampling)
- Vector index 集成 (HNSW / IVF metadata stored in format)
- Schema evolution + multi-version support native
- Multimodal data support (binary / variable-length column)

→ LanceDB 是 wiki 内**唯一 vendor 把 storage format 作 OSS standard** (类似 Iceberg / Delta / Hudi 在 lakehouse 领域).

### 2. Multimodal lakehouse philosophy

[per sources/docs/lancedb/]

LanceDB **不假设 vector retrieval 是核心 workload**——它假设 vector retrieval 是 multimodal data platform 的一个 capability, 与 OLAP query / feature engineering / ML training 一体.

**典型 case**:
- Training data prep: 通过 SQL filter 选择 batch, vector search 找 similar samples, 一同 yield to ML training
- Feature engineering: vector embedding + structured features 同 table SQL-query
- Production retrieval: vector search + filter + raw image return

→ vs Pinecone "pure vector retrieval" / Turbopuffer "first-stage retrieval" / Chroma "RAG dev experience": LanceDB **dimensionally different**——是 lakehouse 而非 vector DB.

### 3. OSS embedded + Enterprise 双模式 (类似 Chroma)

vs Chroma OSS Core + Cloud:
- Chroma OSS Core = single-node HNSW; Cloud = Distributed SPANN — **不同 codebase**
- LanceDB OSS = Lance format embedded; Enterprise = distributed Lance format — **same format core**

→ LanceDB 双模式架构连续性更强 (相同 Lance format), Chroma 有 OSS → Cloud architectural break.

### 4. GPU index building —— claim 待核实（2026-05 修正）

⚠️ **此节为 2026-05-12 首轮 ingest 据 docs.lancedb.com 摘要写下的 claim,2026-05-21 深化抓取无法印证,降级为待核实。**

- 2026-05-12 [per sources/docs/lancedb/README.md WebFetch 摘要]:记 "GPU support for vector index building",据此 wiki 曾断言 LanceDB 是"唯一明示 GPU index 构建的 OSS vendor"。
- 2026-05-21 [per sources/docs/lancedb-2026-05/lancedb-deepened.md]:深化抓取 `indexing/vector-index.md` + `indexing/quantization.md` **完全未提 GPU index build / CUDA**。
- **结论**:GPU build 可能 (a) 已 deprecated、(b) 文档在别处(未抓到的 page)、(c) 首轮摘要误记。**在核实前不再作为 LanceDB 确定卖点**。Milvus 的 GPU index([CAGRA](./cagra.md) via NVIDIA RAFT)仍是 wiki 内唯一 source-confirmed 的 OSS GPU index 路径。

> **lint 教训**:首轮 docs 摘要(非全文)产生的断言,深化抓取一手页面后**反被证伪/无法印证**——这正是 [topics/ann-benchmarking-methodology.md](../topics/ann-benchmarking-methodology.md) "benchmarks/docs lie" 精神在 doc ingest 上的体现:摘要 ≠ 一手,断言须可追到具体页。

### 5. SQL-first query interface

LanceDB SQL filter + vector search:
```sql
SELECT * FROM products
WHERE category = 'wallet' AND price < 500
ORDER BY embedding <-> $query_vector
LIMIT 10
```

vs other vendor:
- pgvector: SQL native (但需 Postgres)
- LanceDB: **SQL native standalone** (no DBMS prerequisite)
- Vespa: YQL (specific dialect)
- Others: API-based

→ LanceDB 是 wiki 内**第 2 个 SQL-native standalone vendor** (vs pgvector requires Postgres).

### 6. ML framework integration depth

vs others:
- Chroma LangChain / LlamaIndex via vendor-bundled SDK
- Milvus LangChain / LlamaIndex via community connector
- **LanceDB: LangChain / LlamaIndex / DuckDB / Pandas / Polars 等 native first-class**——更深的 data engineering ecosystem integration

→ LanceDB 是 wiki 内 **ML / data engineering ecosystem integration 最深** vendor.

## 2026-05 深化：Enterprise 架构 + 完整索引族 + Geneva

[per sources/docs/lancedb-2026-05/lancedb-deepened.md] —— 以下补足 2026-05-12 partial ingest 缺的内核细节。

### A. Enterprise 3-plane disaggregation（object storage primary）

首轮只知 LanceDB Enterprise 是"distributed managed",架构是黑盒;深化抓 `enterprise/architecture.md` 后清楚了:

```
┌──────────────── Control Plane ────────────────┐
│ config / service discovery / identity / policy │
│ / cluster lifecycle                            │
└────────────────────────────────────────────────┘
┌──────────────── Data Plane ───────────────────┐
│ query nodes     —— client-facing: 校验+plan+返回 │
│ plan executors  —— read-execution: cache-backed  │
│                    reads against object storage  │
│ indexers        —— 后台: build / merge / compact │
│   (三者独立 scale, 不抢同一 compute)             │
└────────────────────────────────────────────────┘
┌──────────────── Object Storage ───────────────┐
│ table data + manifests + index artifacts        │
│ "durable record lives outside any query node"   │
└────────────────────────────────────────────────┘
```

→ **LanceDB Enterprise 是明确的 object-storage-primary disaggregated 架构**,与 [Turbopuffer](./turbopuffer.md)（object storage 唯一 stateful 依赖）、[Chroma](./chroma.md) Cloud、[Databricks](./databricks-vector-search.md) 同一哲学族。关键卖点:**request handling / read execution / index-building 三者独立伸缩**——"query fleets scale for interactive traffic without also scaling background indexing"。一致性模型 + 多级缓存细节 docs 未明示(仅 plan executor 的 cache-backed reads)。

### B. 完整向量索引族（首轮只抓到 "IVF + HNSW"）

[indexing/vector-index.md, indexing/quantization.md]

| 索引 | 说明 | 关键参数起点 |
|---|---|---|
| IVF_FLAT | raw vector 无量化 | `num_partitions = num_rows//4096` |
| **IVF_PQ**（默认量化） | dim ≤256 时常优于 IVF_RQ | `num_sub_vectors = dim//8` |
| **IVF_RQ** | **RaBitQ-style,1 bit/dim,极强压缩** | `num_bits` 默认 1 / `sample_rate` 256 / `max_iterations` 50 |
| IVF_SQ | scalar quantization | — |
| IVF_HNSW_FLAT | 最高 recall 无量化 | `num_partitions = num_rows//1048576`, `ef_construction` 150 |
| **IVF_HNSW_SQ** | **best recall/latency trade-off** | 同上 + SQ |
| IVF_HNSW_PQ | IVF partition + HNSW graph + PQ | 同上 + PQ |
| binary | 仅 **IVF_FLAT + hamming** | — |

- 距离:l2(默认)/ cosine / dot / hamming。Multivector(ColBERT-style)当前要求 cosine。
- Search 旋钮:`nprobes`(默认 auto-tune)+ `minimum/maximum_nprobes`(filter 激活时先扫 min,不够 limit 再扩到 max——**这是 filter-aware 自适应扩展,类似 [pgvector iterative scan](./pgvector.md) 哲学**)+ `ef`(1.5k→10k)+ `refine_factor`(多读候选内存重排)。

### C. RaBitQ 进入 production —— IVF_RQ

**LanceDB 的 IVF_RQ = [RaBitQ](../concepts/rabitq.md)（"1 bit per dimension"）。** 这是 RaBitQ 从学术 SOTA 走向 production 的明确 vendor 锚点之一(另一是 Milvus 的 IVF_RABITQ index)——补上了 [concepts/rabitq.md](../concepts/rabitq.md) 此前把"vendor 采用"列为 *logical next step* 的空白。实测压缩:**1024-d float32 4KB → ~几百 bytes**(与 RaBitQ 论文 D-bit ≈ 一半 PQ 码长的 claim 一致)。

### D. 5-tier storage latency 模型

[storage/index.md] —— **immutable fragments** 是存储原语(→ stateless 水平扩展):

| 后端 | p95 延迟 | 备注 |
|---|---|---|
| Object Storage (S3/GCS/Azure) | hundreds of ms | unlimited 但 **QPS bound by concurrency** |
| File Storage (EFS/Filestore) | <~100ms | |
| Third-party (MinIO/WekaFS) | <100ms | |
| Block (EBS/GCP) | <30ms | **not shareable across instances** |
| Local (SSD/NVMe) | <10ms | not shareable |

→ 对应上一轮 [queries/milvus-laion-100m-ingest-rate.md](../queries/milvus-laion-100m-ingest-rate.md) 的存储介质讨论:object storage 是 source of truth + 容量无限但 QPS 受 concurrency limit 约束,这是 object-storage-primary 架构的共性瓶颈。

### E. Enterprise benchmark（⚠️ vendor 自测）

[enterprise/benchmarks.md] —— **LanceDB 自家发布,self-published,须按 [topics/ann-benchmarking-methodology.md](../topics/ann-benchmarking-methodology.md) "benchmarks lie" 第二陷阱(厂商自测偏向)处理**:

- 数据集:dbpedia-openai **1M × 1536d** + synthetic **15M × 256d**
- Vector search(warmed cache):**P50 25ms / P99 35ms / max 49ms**
- + 选择性 filter:P50 30ms / P99 50ms;+ 宽 filter:P50 65ms / **P99 100ms**(宽 filter 把 P99 拉高 2×——印证 [attribute-filtering](../topics/attribute-filtering.md) selectivity 拐点)
- FTS:P50 26ms / P99 42ms
- "thousands of QPS in some deployments";**无 recall / 无 ingestion rate / 无硬件规格 / 无对比系统**——典型 vendor benchmark 的信息缺口。

### F. Geneva —— 多模态 feature engineering（Enterprise-only，NEW）

[geneva/index.md] 首轮未捕获的新组件:把 Python **UDF 作为 Lance table 的 virtual column**(prototype → UDF decorator → `Table.add_columns()` 注册 → backfill),执行可落 **本地 / Ray / KubeRay**。意义:**把 feature engineering 内嵌进 vector DB 的存储层**——这是 LanceDB "lakehouse-first" 哲学的具体落地(feature 计算不是外挂 pipeline 而是 table 的 virtual column),wiki 内**唯一 vendor 把分布式 feature engineering 作 first-class**。

### G. 索引版本语义（reindexing —— 回答"索引是否多版本"）

[per sources/docs/lancedb-2026-05/reindexing-versioning.md] —— 2026-05-25 补抓 `indexing/reindexing.md` + `tables/versioning.md`。

**结论:LanceDB 的"多版本"是数据层的;索引不是"每个数据版本一份可时间旅行的快照",而是单一增量演化索引。**

- **数据/表:多版本 ✓**——update/add/delete 产生 version,`checkout(version)` / `restore()` 快速回滚 without data duplication。
- **索引:增量并入,非全量重建**——`optimize()` 把新数据 "adds newly-ingested data to **existing** vector/scalar/FTS indexes" + compaction + cleanup(**Enterprise 自动 / OSS 手动**)。
- **reindex 前的新数据照样可查**——LanceDB "combine results from the existing index with **exhaustive/flat search on the new data**":不漏数据,但未索引数据越多 latency 越高。
- **索引更新被记入版本号**——"`optimize()`, index updates, and table compaction **also increment table version numbers**":索引变更是版本时间线上的事件。
- **但 per-version 索引快照 = docs 明确未覆盖**——版本是否捕获 index、checkout 老版本用哪一版索引,`reindexing.md` 与 `versioning.md` **都 explicitly 不回答**;且**旧文件版本默认 7 天后 prune**——这强烈暗示**不能可靠地对索引做远期 time-travel**。

→ 一句话给用户:**"索引多版本"在 LanceDB ≈ 不成立**。索引是**单一、随 `optimize()` 增量合并**的对象,其更新虽然会 bump version number,但 LanceDB 不承诺"每个数据版本各自冻结一份可回溯的索引",且老版本默认 7 天回收。要"老数据版本 + 当时的索引"一起 time-travel,docs 无 source 支撑。

### H. 原生空间索引（R-Tree，2026-02 新增）

[per sources/docs/lance-geo-2026-02/geo-support.md] —— **纠正此前判断**:2026-05-21/25 深化基于更早 docs 快照得出"LanceDB 无原生 spatial 索引";**Lance 已于 2026-02-25 加入原生地理空间支持**(blog "How We Added Geospatial Support To Lance With No New Code")。是 Lance(格式/引擎)层,LanceDB 继承。三块:

1. **真正的 R-Tree 空间索引(production-ready)**:static/immutable 2D,bounding-box,多层 hierarchical(leaf `(bbox,rowid)` / branch 子 bbox 聚合 / 单 root);**packed-build + Hilbert 曲线排序**;剪枝按 `ST_Intersects(geometry, query_bbox)` 从 root 逐层 descend/prune subtree;**需显式建索引**;由 **ByteDance Xin Sun** 贡献。
2. **GeoArrow 扩展类型**:Point/LineString/Polygon/Multi*/GeometryCollection + CRS——这块是 "with no new code"(Arrow extension type 机制原生,**仅存储白嫖**)。
3. **GeoDataFusion 空间函数**(OGC Simple Feature Access):ST_Distance/Intersects/Contains/Within/Touches/Crosses/Overlaps/Covers/CoveredBy——**新写的集成代码**。

> **wiki 影响**:**LanceDB 成为 wiki 内第二个有原生 spatial 索引的 vendor**(继 [Vespa](./vespa.md) 之后),且是明确的 **R-Tree**(比 Vespa 的 R-tree-like 更具体)。这部分填补 wiki 长期"空间能力仅 Vespa native"的盘点空白——详见 [topics/multimodal-embedding-retrieval.md](../topics/multimodal-embedding-retrieval.md)。**但注意:这是 algorithm/index 能力,不是 benchmark**——vector+spatial 的公平横测 benchmark 仍空白(见 [queries/hybrid-retrieval-benchmark-landscape.md](../queries/hybrid-retrieval-benchmark-landscape.md))。**限制**:R-Tree 需显式建;Spark/Trino/DuckDB/Ray 引擎集成 + HF geo 数据集仍 future work。

## Scale 边界

[per sources/docs/lancedb/, sources/docs/lancedb-2026-05/lancedb-deepened.md]

| Metric | LanceDB |
|---|---|
| OSS Core scale | up to billions of vectors single node |
| Enterprise scale | petabyte-scale (multimodal: video + point cloud + image) |
| Enterprise 实测 latency (vendor 自测) | 1M×1536d / 15M×256d: vector P99 35ms, +宽 filter P99 100ms |
| GPU index building | **待核实**（2026-05 indexing docs 未提，见 §4 + Open Q） |
| Polyglot SDK | Python / TypeScript / Java / Rust |
| Embedded library mode | ✓ (类似 SQLite for vector + multimodal) |
| Multi-region | Enterprise only |

### 瓶颈

- **Mid-tier hybrid retrieval ergonomics**: 较 Chroma `Rrf()` / Weaviate `hybrid()` 更基础 (依赖 SQL + vector search 手动 fusion)
- **Sparse vector support**: Lance format 当前 dense vector 主流, sparse 支持 less explicit (vs Chroma SparseVectorIndexConfig / pgvector sparsevec)
- **ACORN / filter-aware ANN**: 未明示 first-class (vs Qdrant ACORN / Vespa Acorn-1 / pgvector iterative_scan)
- **AI-agent protocol integration (MCP)**: 未明示 native (vs Chroma MCP first-class)

## 生产案例

[per LanceDB website + community]

- **Multimodal AI training**: 大规模 image / video / point cloud dataset management
- **Feature engineering at scale**: Petabyte-scale feature stores
- **High-performance retrieval applications**: dev embedded + Enterprise managed dual deployment
- **Specific customer 名单**: 不公开

## Open Questions

- **Lance format vs Parquet/Arrow performance**: 实测 random access / vector index integration 性能差异 不公开
- **Lance format 跨 ecosystem 支持**: Iceberg / Delta / Hudi 是否 future integration?
- **LanceDB OSS 到 Enterprise migration**: same Lance format core, migration cost vs Chroma OSS-to-Cloud cost
- **GPU index building 是否真存在**: ⚠️ 2026-05-12 摘要记为支持,2026-05-21 深化抓 indexing/quantization docs **未提** → claim 待核实(deprecated? 别处文档? 摘要误记?)。在确认前不应宣称 LanceDB 有 GPU index build;CAGRA via Milvus 仍是 wiki 内唯一 source-confirmed OSS GPU index 路径
- **IVF_RQ (RaBitQ) 实测 vs IVF_PQ**: LanceDB docs 给 dim≤256 时 IVF_PQ 常优于 IVF_RQ 的定性,但无 head-to-head recall/QPS 数;[RaBitQ 论文](../concepts/rabitq.md)的 6/6 dominate 是独立实现,LanceDB IVF_RQ 实测未公开
- **Lance format 多模态 storage cost vs separate object storage**: 实际 cost 对比不公开
- **LanceDB Enterprise petabyte-scale production case**: 公开 customer scale 数据 不存在
- **SQL query optimization**: LanceDB SQL planner 性能 vs DuckDB / 其他 OLAP engine 不公开
- **Sparse vector + Lance format**: sparse vector representation in Lance format 不深入
- **Hybrid retrieval native API**: LanceDB 当前依赖 SQL filter + vector search 手动 fusion, native RRF API 是否 future?
- **MCP / AI agent protocol integration**: 未明示 native vs Chroma first-class
- **Multi-version embedding migration**: zero-copy add column 模式 vs Chroma fork 模式 head-to-head。**2026-05-25 部分澄清(见 §G)**:数据层多版本 + 索引层单一增量(optimize 合并)已 source-confirmed;**仍 open**——per-version 索引快照 / checkout 老版本时的索引语义 docs 明确未覆盖,且旧版本默认 7 天 prune,索引远期 time-travel 无 source
- **vs Milvus 在 multimodal + GPU 重叠领域**: Milvus 通过 CAGRA + 多 vector field 接近 LanceDB capability, head-to-head 不公开

Cited by: 待 query 引用
