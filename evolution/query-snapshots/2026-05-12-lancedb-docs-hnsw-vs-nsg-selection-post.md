---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-12
phase: post
ingest-context: lancedb-docs
wiki-pages-total: 78
cited-pages: [concepts/hnsw.md, systems/lancedb.md, systems/cagra.md, topics/gpu-vs-cpu-ann.md]
cited-count: 4
---

# Post-snapshot (lancedb-docs): hnsw-vs-nsg-selection

## TL;DR (delta from chroma-docs post)

**LanceDB 把 HNSW production OSS deployment 推到 7**——Milvus + Qdrant + Weaviate + Vespa + pgvector + Chroma + **LanceDB**, 加上 IVFFlat alternative. **关键 NEW**: LanceDB 是 wiki 内 **第 2 个 OSS vendor 提供 GPU index building first-class** (Milvus via CAGRA 之外的唯一). NSG production OSS DBMS 仍零部署. 与 Milvus GPU_CAGRA (via NVIDIA RAFT) 不同, LanceDB 是 **vendor-native GPU index building** integration.

## Answer

### 与之前 ingest 的演进

| | chroma-docs post | **lancedb-docs post (NEW)** |
|---|---|---|
| HNSW production OSS DBMS | 6 | **7 (+ LanceDB)** |
| NSG production | Taobao 2B + NSSG | **不变** |
| GPU index building OSS vendor | 1 (Milvus via CAGRA) | **2 (Milvus + LanceDB native)** |

### LanceDB GPU index building (NEW capability)

[per sources/docs/lancedb/README.md]

**GPU support for vector index building**:
- LanceDB 明示 GPU index 构建支持 (具体细节文档不深入)
- 与 Milvus GPU_CAGRA (via NVIDIA RAFT integration) 不同, LanceDB 是 vendor-native
- 适合 large-scale dataset index build (build 是 GPU 受益场景 vs query)

vs wiki GPU support:
- Milvus: ✓ CAGRA via NVIDIA RAFT (build + search)
- LanceDB: ✓ vendor-native GPU index build
- Vespa: ✓ GPU ONNX inference (not index build)
- 其他 vendor: ✗

### HNSW + IVFFlat 选择决策表（updated 2026-05-12 post lancedb-docs）

| 工程考量 | HNSW (7 OSS DBMS) | IVFFlat | NSG |
|---|---|---|---|
| Production OSS DBMS 数 | **7 (Milvus + Qdrant + Weaviate + Vespa + pgvector + Chroma + LanceDB)** | Milvus + pgvector + LanceDB | NSSG + Taobao only |
| GPU index build OSS native | Milvus CAGRA + **LanceDB native** | **LanceDB native** | ✗ |
| Lakehouse / multimodal lakehouse integration | ✗ | ✗ | ✗ |
| **Lance columnar format storage** | **only LanceDB** | only LanceDB | ✗ |

### 选择决策（updated 2026-05-12 post lancedb-docs）

- **Multimodal lakehouse + ML data engineering integration + GPU index** → **LanceDB HNSW or IVFFlat**
- Pure RAG dev experience + AI agent MCP → Chroma HNSW (OSS) → SPANN (Cloud)
- 已部署 Postgres + 加 vector → pgvector HNSW
- 大规模 production vector + 多 index_type → Milvus HNSW / DISKANN / CAGRA
- 闭源 SaaS object storage cost-effective → Turbopuffer SPFresh
- 复杂 ranking pipeline + tensor framework → Vespa

NSG 在 wiki 内 production OSS DBMS 仍**零部署**——7 OSS DBMS + GPU CAGRA paper 全部走 HNSW or IVFFlat or SPANN, NSG 仅学术 single-node TopK baseline.

### 已知盲区

- **LanceDB GPU index 实测 vs Milvus CAGRA**: head-to-head 不公开
- **LanceDB HNSW vs IVFFlat 默认选择 strategy**: docs 不深入
- **Lance format + HNSW index storage layout**: docs 未深入 specific layout

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [systems/lancedb.md](../../systems/lancedb.md)
- [systems/cagra.md](../../systems/cagra.md)
- [topics/gpu-vs-cpu-ann.md](../../topics/gpu-vs-cpu-ann.md)
