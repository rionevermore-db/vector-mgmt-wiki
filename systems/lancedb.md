---
title: LanceDB（multimodal lakehouse for AI, OSS embedded + Enterprise）
type: system
sources: [lancedb-docs]
related: [chroma.md, pgvector.md, milvus.md, faiss.md, turbopuffer.md, pinecone.md, ../concepts/hnsw.md, ../concepts/product-quantization.md, ../concepts/cagra-graph.md, cagra.md, ../topics/gpu-vs-cpu-ann.md, ../topics/disk-vs-memory-ann.md, ../topics/index-selection.md, ../topics/multimodal-embedding-retrieval.md, ../topics/sparse-dense-hybrid-retrieval.md]
created: 2026-05-12
updated: 2026-05-12
---

# LanceDB

**TL;DR**: LanceDB 是 **"multimodal lakehouse for AI"** 哲学的 vector DBMS, 由 Chang She + Lei Xu 2022 创立 (Series A 2024). OSS Apache-2.0 embedded library (Python / TypeScript / Java / Rust) + LanceDB Enterprise (distributed managed multimodal lakehouse). **对 wiki 内 vector DBs 的核心独特性**: (1) **Lance columnar format = 独特存储 axis**——所有 wiki 内其他 vector DBMS 都用 vendor-specific storage (Milvus segment / Qdrant single binary / Chroma SQLite / pgvector Postgres heap / Vespa tensor / Pinecone slab / Turbopuffer object storage); LanceDB 用 **OSS Lance columnar format** (类似 Parquet 但 ML-optimized + vector index integration)——**wiki 内首个 vendor 以 OSS columnar format 作 primary storage substrate**, 与 Iceberg / Delta / Hudi 这类 lakehouse format philosophy 并列; (2) **Multimodal lakehouse 哲学**——单一 table 同时存 vector + metadata + 原始 multimodal data (text / image / video / point cloud) + 版本控制 + zero-copy ops——其他 vector DBs 普遍 vector + metadata only, raw data 推到 separate object storage; (3) **OSS embedded library + Enterprise 双模式** (类似 Chroma)——OSS dev / Enterprise petabyte-scale managed, 但**架构 unique**: 基于 Lance format 不切换 codebase (vs Chroma OSS Core HNSW → Cloud SPANN 不同 codebase); (4) **GPU index building support**——wiki 内 OSS vector DB 中**唯一明示 GPU index 构建**支持的 vendor (其他用 CAGRA via Milvus 但 LanceDB 直接集成); (5) **SQL query support + ML framework integration**——同 platform 内支持 LangChain / LlamaIndex / DuckDB / Pandas / Polars 直接 query, **超越纯 vector DB 边界进入 data engineering / feature engineering 领域**; (6) **Zero-copy versioning**——schema evolution + add new columns 不需 copy existing data, "table-level git-like" version control; (7) **Polyglot SDKs**: Python / TypeScript / Java / Rust 4 语言一等公民, Rust core + 多语言 binding. **Position**: 与 Chroma 同 "OSS embedded + Cloud" 双模式, 但走 **lakehouse-first** 而非 RAG-dev-first 路线. [per sources/docs/lancedb/]

## 与 wiki 内其他 system 的定位差异

| | Milvus | Qdrant | Weaviate | Vespa | Pinecone | Turbopuffer | pgvector | Chroma | **LanceDB** |
|---|---|---|---|---|---|---|---|---|---|
| Storage substrate | segment (custom) | single Rust binary | Go binary | tensor + custom | slab (闭源) | object storage primary | Postgres heap | SQLite (OSS) / object storage (Cloud) | **Lance columnar format (OSS)** |
| Storage format 是否 OSS standard | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | Postgres native | partial | **✓ Lance OSS format** (类比 Iceberg/Delta/Hudi) |
| Embedded library 模式 | ✗ (DBMS only) | ✗ (DBMS) | ✗ (DBMS) | ✗ (engine) | ✗ (SaaS only) | ✗ (SaaS) | extension | **✓ (OSS Core)** | **✓ (OSS) — primary mode** |
| Multimodal raw data + vector 同 table | partial (多 vector field) | multiple vectors per point | named vectors | tensor + content | 不公开 | namespace-per-asset | sparsevec / vector type only | text + metadata + vector | **✓ vector + metadata + raw multimodal (image / video / point cloud)** |
| GPU index build | ✓ via CAGRA (Milvus integration) | ✗ | ✗ | ✗ | 不公开 | ✗ | ✗ | ✗ | **✓ first-class GPU index building** |
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
              │  Index (IVF / HNSW / GPU build)       │
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

### 4. GPU index building first-class

[per sources/docs/lancedb/README.md]

wiki 内 vendor GPU support 情况:
- Milvus: ✓ via CAGRA (NVIDIA RAFT integration)
- Qdrant: ✗
- Weaviate: ✗
- Vespa: ✓ GPU ONNX inference (not index build)
- Pinecone: 不公开
- Turbopuffer: ✗
- pgvector: ✗
- Chroma: ✗
- **LanceDB**: ✓ **first-class GPU index building**

→ LanceDB + Milvus 是 wiki 内 GPU index 主流 OSS vendor.

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

## Scale 边界

[per sources/docs/lancedb/]

| Metric | LanceDB |
|---|---|
| OSS Core scale | up to billions of vectors single node |
| Enterprise scale | petabyte-scale (multimodal: video + point cloud + image) |
| GPU index building | first-class |
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
- **GPU index building 实测 vs CAGRA (NVIDIA RAFT)**: head-to-head benchmark 不公开
- **Lance format 多模态 storage cost vs separate object storage**: 实际 cost 对比不公开
- **LanceDB Enterprise petabyte-scale production case**: 公开 customer scale 数据 不存在
- **SQL query optimization**: LanceDB SQL planner 性能 vs DuckDB / 其他 OLAP engine 不公开
- **Sparse vector + Lance format**: sparse vector representation in Lance format 不深入
- **Hybrid retrieval native API**: LanceDB 当前依赖 SQL filter + vector search 手动 fusion, native RRF API 是否 future?
- **MCP / AI agent protocol integration**: 未明示 native vs Chroma first-class
- **Multi-version embedding migration**: zero-copy add column 模式 vs Chroma fork 模式 head-to-head
- **vs Milvus 在 multimodal + GPU 重叠领域**: Milvus 通过 CAGRA + 多 vector field 接近 LanceDB capability, head-to-head 不公开

Cited by: 待 query 引用
