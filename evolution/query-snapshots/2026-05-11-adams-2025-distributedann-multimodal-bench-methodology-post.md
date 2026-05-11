---
query-key: multimodal-bench-methodology
query: "对比多个 vector DB 在向量 + 标量 + 空间三模检索下的性能，怎么公平 benchmark？关键挑战：空间能力严重不对称（真索引 / bbox 近似 / 完全没有），怎么处理？"
date: 2026-05-11
phase: post
ingest-context: adams-2025-distributedann
wiki-pages-total: 69
cited-pages: [systems/distributedann.md, systems/vespa.md, systems/turbopuffer.md, topics/attribute-filtering.md]
cited-count: 4
---

# Post-snapshot (adams-2025-distributedann): multimodal-bench-methodology

## TL;DR (post-ingest of adams-2025)

DistributedANN paper **不涉及空间维度** — Bing web search 三模查询 (vector + scalar + spatial) 不是论文 focus。**关键 NEW**: DistributedANN 给 wiki **spatial benchmark methodology 一个间接 insight**——production-scale search engine (Bing) 公开**仅 focus 在 vector + scalar**，**spatial 在 Bing web 搜索可能不是核心**（web 搜索的 spatial 多通过 attribute filter "city=X" 而非真 geo index）。这与 wiki 之前 multimodal frontier 空白一致——所有 OSS vector DBMS 内**只有 Vespa 有真 native spatial index**。3 模公平 benchmark 仍是 **wiki industry coverage gap**.

## Answer

### Wiki 当前 7 vendor 空间能力 inventory（NEW counting）

| Vendor | Geo 字段 | 空间索引 | 距离计算 | Bbox 查询 | 真 spatial 路径 |
|---|---|---|---|---|---|
| Milvus | `float` lat/lng | ✗ | application | application | ✗ |
| Qdrant | `geo_point` / `geo_polygon` | geohash-aware filter | docs 提 | bbox filter | 半 native |
| Weaviate | `geoCoordinates` | ✗ (filter only) | application | bbox filter | ✗ |
| Pinecone | `geo` filter | 不公开 | 不公开 | bbox filter | 不公开 |
| Vespa | `position` (2D) | **GeoSearch + position dimension** | native great-circle | bbox filter | **✓ native** |
| Turbopuffer | `float` lat/lng | ✗ | application | application | ✗ |
| **DistributedANN (NEW)** | **paper 不涉及** | **paper 不涉及** | **paper 不涉及** | **paper 不涉及** | **paper 不涉及** |

→ DistributedANN production focus 是 Bing web search 的 vector retrieval。Bing 整体当然有 spatial search (e.g., 地图、本地 POI), 但论文 scope 仅 vector ANN over web docs.

### DistributedANN 间接 insight on spatial methodology（NEW）

[paper 不涉及 spatial; 但 Bing web search 实际 production 推测]

**Insight 1**: 大型 web search engine (Bing 公开 architecture) 把 vector retrieval + spatial 分离:
- Vector retrieval (DistributedANN paper focus): semantic relevance via dense embeddings
- Spatial filter: 通过 attribute filter "country=US", "geoip=X", etc. (paper 不深入)
- 真 spatial geo-index 可能在 retrieval pipeline 的不同阶段 (Bing 整体 indexer)

**Insight 2**: DistributedANN architecture 的"orchestration service + node scoring service" 是 vector-only — spatial integration 在更上层 (web search 的 ranking pipeline) 处理.

→ 对 wiki "3 模 fair benchmark" 这个 query 的隐性 takeaway: **production-scale system 倾向于 separate 各 modality + 在更高层 pipeline 融合**，**而非 single ANN system 内集成所有 modality**. Vespa 是反例 (single system + position dimension)，但 Vespa 也是 web search engine lineage —— **没有 vector DBMS native 把 dense vector + spatial 一等公民集成的成功 production 案例**.

### 3 模公平 benchmark 框架（updated 2026-05-11 post adams-2025）

**Tier 1: 设计选择 inventory**:
1. **Native spatial integration** vs **multi-system pipeline**:
   - Native (Vespa): single system, 三 modality 一等公民
   - Multi-system (Bing implied): separate vector ANN system + separate geo system + 上层 pipeline 融合
   - 大部分 OSS vector DBMS: 仅支持 attribute filter, 不算 native spatial

2. **Workload representativity**:
   - "POI search 餐厅 + Italian + 5km" 是 Vespa native 占优 workload
   - "Web search vector + geo" 是 Bing multi-system 占优 workload
   - 两类 workload 公平比较需 normalize

3. **Application-fanout pattern (most OSS DBMS)**:
   - Vector ANN top-K' → 应用 filter bbox + scalar
   - OR spatial bbox 先 filter → 候选集中 ANN
   - 选 best of two paths per vendor

**Tier 2: 公平 benchmark protocol**:
- 真 spatial vendor (Vespa): single API call, planner 决定
- Geo-aware filter vendor (Qdrant): vector + geohash filter, 一次 API
- Pure attribute filter (Milvus / Weaviate / Pinecone / Turbopuffer / **DistributedANN-like**): application-fanout, 报 app overhead 单独
- **DistributedANN 等大规模 single-system 不涉及 spatial** → workload 假设排除 这类 system on 3-modal queries

### Wiki industry coverage gap (unchanged)

- **Native R-tree / Quadtree in vector DBMS**: 6 OSS + 1 闭源 (Pinecone) + 2 production-paper-documented (DistributedANN, Turbopuffer) 全无 (Vespa "position" is 2D point heritage, simpler than R-tree)
- **三 way correlation optimization algorithm**: zero coverage
- **Spatial + Streaming / DistributedANN / object-storage combo**: zero coverage
- **3D / N-D spatial (球面 / 卫星 / 气象)**: zero coverage

### 已知盲区

- **Bing spatial pipeline (不在 DistributedANN scope)**: paper 不涵盖 Bing 整体 architecture
- **DistributedANN + spatial filter**: technically attribute filter 可加 (geo-as-attribute), 但 paper 不讨论
- **3 模 head-to-head**: 仍无 fair benchmark
- **Production-scale vector + spatial native**: Vespa 是唯一; 大规模 Bing-scale 实际 production 未见

## Cited Pages

- [systems/distributedann.md](../../systems/distributedann.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
