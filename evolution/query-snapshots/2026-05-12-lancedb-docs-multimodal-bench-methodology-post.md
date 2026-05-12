---
query-key: multimodal-bench-methodology
query: "对比多个 vector DB 在向量 + 标量 + 空间三模检索下的性能，怎么公平 benchmark？关键挑战：空间能力严重不对称（真索引 / bbox 近似 / 完全没有），怎么处理？"
date: 2026-05-12
phase: post
ingest-context: lancedb-docs
wiki-pages-total: 78
cited-pages: [systems/lancedb.md, topics/multimodal-embedding-retrieval.md]
cited-count: 2
---

# Post-snapshot (lancedb-docs): multimodal-bench-methodology

## TL;DR (post-ingest of lancedb-docs)

**LanceDB 让 wiki "multimodal" 一词的 2 种含义 (cross-modal embedding vs tri-modal vector+scalar+spatial) 之外新增第 3 种含义**: **multimodal data lakehouse** (vector + metadata + raw image / video / point cloud / text 同 table 存储). **关键 NEW**: LanceDB 是 wiki 内**唯一明示存原始 multimodal data (image binary / video frames / point cloud) + vector embedding + metadata 同 schema** 的 vendor. 其他 vector DBs 普遍仅存 embedding + metadata, raw binary 推到 separate object storage. 这是**multimodal lakehouse-level benchmark** 全新 axis.

## Answer

### "Multimodal" 一词在 wiki 内 3 种含义（NEW clarification）

[per radford-2021-clip + topics/multimodal-embedding-retrieval.md + LanceDB lakehouse philosophy]

1. **Cross-modal embedding**: CLIP / SigLIP / ImageBind 等 joint embedding space (text + image + audio 共享 vector space)——algorithm-level multimodal
2. **Tri-modal query**: vector + scalar + spatial 三种 query type 同 retrieval (Vespa native + pgvector + PostGIS)——query-level multimodal
3. **NEW: Multimodal data lakehouse**: 存原始 multimodal data (image / video / point cloud / text) + embedding + metadata 同 table——**storage-level multimodal** (LanceDB unique)

### Vendor multimodal storage 能力 inventory (NEW comprehensive)

[per LanceDB + 其他 vendor docs]

| Vendor | Vector + metadata | Raw multimodal (image / video / point cloud) storage |
|---|---|---|
| Milvus | ✓ 多 vector field | ✗ (推到 separate object storage) |
| Qdrant | ✓ | ✗ |
| Weaviate | ✓ named vectors | ✗ (image text reference only) |
| Vespa | ✓ tensor framework + ONNX | partial (binary field 存有限) |
| Pinecone | ✓ | 不公开 |
| Turbopuffer | ✓ | ✗ (推到客户 object storage) |
| pgvector | ✓ | partial via PG `bytea` (实际不常用) |
| Chroma | ✓ + Chroma Sync ingest | ✗ (推到 chroma sync source) |
| **LanceDB** | **✓** | **✓ first-class native (image binary / video / point cloud column)** |

→ LanceDB **唯一明示 storage-level multimodal native**——multimodal lakehouse 独有 axis.

### 4-axis multimodal benchmark methodology (updated 2026-05-12 post lancedb-docs)

之前 axis (per chroma-docs post):
1. Cross-modal embedding
2. Tri-modal query (vector + scalar + spatial)
3. AI-agent retrieval workload integration (MCP)

**NEW axis 4**:
4. **Storage-level multimodal** (raw binary + vector + metadata 同 schema): LanceDB unique

### Vespa vs LanceDB 多模态 frontier 对比 (updated)

| | Vespa | **LanceDB** |
|---|---|---|
| Cross-modal embedding (CLIP) native | ✓ tensor + ONNX inline | ✓ via Lance format dense vector |
| Tri-modal (vector + scalar + spatial) | ✓ position dim + tensor + BM25 | partial (no native spatial) |
| Storage-level multimodal (raw + vector) | partial | **✓ first-class** |
| ML ecosystem integration (LangChain / DuckDB / etc.) | 应用层 | **✓ native** |

→ Vespa 是 retrieval-side 多模态 native 主流, LanceDB 是 storage-side + ML ecosystem 多模态 native 主流. **两者 axis 不同, 互补**.

### 已知盲区

- **LanceDB storage-level multimodal production case at scale**: 公开 case study 不存在
- **Lance format video / point cloud encoding 实测**: 不公开
- **LanceDB vs Iceberg/Delta+vector overlay 多模态 storage 哲学对比**: 不公开

## Cited Pages

- [systems/lancedb.md](../../systems/lancedb.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
