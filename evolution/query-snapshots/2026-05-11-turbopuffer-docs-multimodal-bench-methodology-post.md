---
query-key: multimodal-bench-methodology
query: "对比多个 vector DB 在向量 + 标量 + 空间三模检索下的性能，怎么公平 benchmark？关键挑战：空间能力严重不对称（真索引 / bbox 近似 / 完全没有），怎么处理？"
date: 2026-05-11
phase: post
ingest-context: turbopuffer-docs
wiki-pages-total: 68
cited-pages: [systems/turbopuffer.md, systems/qdrant.md, systems/weaviate.md, systems/milvus.md, systems/pinecone.md, systems/vespa.md, systems/analyticdb-v.md, topics/attribute-filtering.md]
cited-count: 8
---

# Post-snapshot (turbopuffer-docs): multimodal-bench-methodology

## TL;DR (post-ingest of turbopuffer-docs)

空间维度在 wiki 内仍**完全空白**——所有 6 vendor (Milvus / Qdrant / Weaviate / Pinecone / Vespa / Turbopuffer) docs 中**没有任何 native spatial index**（geohash / R-tree / Quadtree）。这是 wiki 内一个**重要的 industry coverage gap**——vector DBMS 普遍把 geo 当作 attribute filter（数值 lat/lng 比较 + bbox 计算），不提供真正的空间索引。**关键 NEW**：Turbopuffer 提供 **glob/regex trigram-based index**——可用于近似空间 pattern matching（e.g., 区域代码、邮编 prefix），但**不是真 spatial index**。3 模检索仍需 application 层 fanout + 客户端 fusion。

## Answer

### 6 vendor 空间能力 inventory（NEW comprehensive）

| Vendor | Geo 字段类型 | 空间索引 | 距离计算 | Bbox 查询 | 真 spatial 路径 |
|---|---|---|---|---|---|
| **Milvus** | `float` lat/lng | ✗ | application | application | ✗ |
| **Qdrant** | `geo_point` / `geo_polygon` (v1.7+) | **geohash-aware filter (precision-level)** | docs 提 distance | bbox filter | **半 native** (geohash filter, 非真 R-tree) |
| **Weaviate** | `geoCoordinates` | ✗ (filter only) | application | bbox via filter | ✗ |
| **Pinecone** | `geo` filter type docs 提 | 不公开 | 不公开 | bbox filter | 不公开 |
| **Vespa** | `position` (2D point) | **GeoSearch + position dimension** (Vespa heritage 优势) | native great-circle | bbox filter | **✓ native**（Yahoo! search lineage 沉淀） |
| **Turbopuffer** | `float` lat/lng (attribute) | ✗ | application | application | ✗ |

→ **Vespa 是 wiki 内唯一有真 native spatial index 的 vector DBMS**——Yahoo! 2003 web search heritage 包含 position dimension。其他全部是 attribute filter 上的 numerical 比较。

### 空间能力 asymmetry 主要分歧

[per topics/attribute-filtering.md 视角]

**3 vendor 路径分歧**:
1. **Native spatial (Vespa)**: position 字段 + GeoSearch 算子（great-circle, bbox, distance order）——与 BM25 / ANN 并列为 retrieval operator
2. **Geohash-aware filter (Qdrant)**: geohash precision-level filter 是 attribute filter 的 specialized variant; 非真 spatial index
3. **Pure attribute filter (Milvus / Weaviate / Pinecone / Turbopuffer)**: lat/lng 作 numerical attribute, bbox 通过 `lat > X AND lat < Y AND lng > A AND lng < B` 计算; 距离 sort 推到 application 层

→ **不公平的根源**：3 路径**根本能力不对等**——Vespa native spatial vs Qdrant geohash-aware vs 其他 attribute filter。

### 公平 3 模 benchmark 框架（vector + scalar + spatial）

**Tier 1: Capability inventory**（不公平的 source）:
1. **Spatial index 类型**: native R-tree / Quadtree / geohash / 无
2. **Distance metric**: great-circle / Euclidean / 应用层
3. **Bbox 查询效率**: native bbox / 通过 4 numerical filter

**Tier 2: 实测协议**:
1. **Test query**: `vector_ann_topK ∧ category=A ∧ within_bbox(lat1,lng1,lat2,lng2)` — 三模组合
2. **Selectivity sweep**: bbox selectivity ∈ {1%, 10%, 50%, 90%}（spatial selectivity 必独立测）
3. **Spatial cardinality**: doc 分布 uniform / clustered / Zipf——影响 spatial index 效率
4. **Three-way correlation**: vector ↔ category ↔ spatial（典型: 商品类 ↔ 颜色 ↔ 城市）

**Tier 3: 公平化 framework**:
- **Native path** (Vespa): single API call → DBMS planner 决定 vector + scalar + spatial 联合执行顺序
- **Geohash-aware path** (Qdrant): vector + spatial filter via geohash —— 一次 API
- **Application-level fanout** (Milvus / Weaviate / Pinecone / Turbopuffer):
  - Option A: ANN top-K' 大候选 → application 端 filter bbox + scalar
  - Option B: spatial bbox 先 filter → 候选集中 ANN
  - **公平 benchmark 需选 best of A/B** per vendor
  - **报 application overhead** 单独——不能混入 DBMS latency

### 实际 case study workload

| Workload | 主流路径 |
|---|---|
| **POI search "餐厅 + Italian + within 5km"** | **Vespa native** (single query, planner 决定) — 唯一无 app overhead 选项 |
| **Real-estate "house + 3BR + downtown"** | Qdrant geohash + ANN（仍可一次 query） |
| **Image search + location tag** | 全部 vendor 都能 (application fanout)，性能差距来自 spatial filter |
| **>1M doc spatial cluster** | Vespa（其他 vendor application fanout 在大规模 spatial 集群 cost 大）|

### Wiki vector DBMS 空间维度 frontier

**未被任何 ingest 关闭的 frontier**:
- **Native R-tree / Quadtree 在 vector DBMS**: 6 vendor 全无（Vespa "position" 是 simpler 2D-point heritage, 不是 R-tree）
- **Geohash precision optimal**: Qdrant precision 控制 trade-off 实测 zero coverage
- **3 模 cross-correlation**: 三 way correlation optimization 算法（类 Weaviate 2 way "positive/negative correlation" 扩展）——wiki zero coverage
- **空间 + Streaming Search 组合**: Vespa Streaming 是 brute-force per-tenant; 加 spatial filter 是否仍经济？docs 未提

### 已知盲区

- **Vespa GeoSearch 实测 vs Qdrant geohash filter 实测**: head-to-head 公开数据 zero
- **Spatial selectivity vs vector ANN crossover**: 何时 spatial filter 主导 vs vector retrieval 主导, 各 vendor 不同
- **Spatial + per-tenant Turbopuffer namespace**: per-namespace docs 通常地理分布有界, 但 docs 未利用 spatial 优化
- **3D / N-D spatial (球面 / 卫星轨道 / 气象)**: 全部 vendor 都是 2D (lat/lng), 高维空间索引 zero coverage

## Cited Pages

- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/qdrant.md](../../systems/qdrant.md)
- [systems/weaviate.md](../../systems/weaviate.md)
- [systems/milvus.md](../../systems/milvus.md)
- [systems/pinecone.md](../../systems/pinecone.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/analyticdb-v.md](../../systems/analyticdb-v.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
