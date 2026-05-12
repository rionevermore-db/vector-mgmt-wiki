---
query-key: multimodal-bench-methodology
query: "对比多个 vector DB 在向量 + 标量 + 空间三模检索下的性能，怎么公平 benchmark？关键挑战：空间能力严重不对称（真索引 / bbox 近似 / 完全没有），怎么处理？"
date: 2026-05-12
phase: post
ingest-context: pgvector-docs
wiki-pages-total: 76
cited-pages: [systems/pgvector.md, topics/multimodal-embedding-retrieval.md, systems/vespa.md]
cited-count: 3
---

# Post-snapshot (pgvector-docs): multimodal-bench-methodology

## TL;DR (post-ingest of pgvector-docs)

**pgvector 给 multimodal / spatial 空白 wiki vendor 提供 surprising candidate**——pgvector + PostGIS (Postgres native spatial extension) 组合是 wiki 内**第 2 个 native spatial + vector 同 system** (vs Vespa position dimension). **关键 NEW**: 之前 wiki 空间能力 inventory 显示**仅 Vespa native spatial**, 其他 7 vendor 都把 geo 当 attribute filter. pgvector + PostGIS 组合通过 Postgres extension 生态**间接 native** spatial: PostGIS 是 Postgres 一等公民 R-tree + GiST 空间索引, 与 pgvector vector index 同 schema 共存. 多模态 embedding (CLIP-style) + spatial + scalar filter + dense vector ANN — **全部通过 Postgres standard SQL 表达**.

## Answer

### Wiki 8 vendor 空间能力 inventory（updated 2026-05-12 post pgvector-docs）

| Vendor | Geo 字段 | 空间索引 | 距离计算 | 真 spatial 路径 |
|---|---|---|---|---|
| Milvus | `float` lat/lng | ✗ | application | ✗ |
| Qdrant | `geo_point` / `geo_polygon` | geohash-aware filter | docs 提 | 半 native |
| Weaviate | `geoCoordinates` | ✗ (filter only) | application | ✗ |
| Pinecone | `geo` filter | 不公开 | 不公开 | 不公开 |
| **Vespa** | **`position` (2D)** | **GeoSearch + position dim** | native great-circle | **✓ native** |
| Turbopuffer | `float` lat/lng | ✗ | application | ✗ |
| DistributedANN | paper 不涉及 | paper 不涉及 | paper 不涉及 | paper 不涉及 |
| **pgvector (NEW)** | **PostGIS `geometry` / `geography`** | **PostGIS R-tree / GiST (Postgres ecosystem)** | **native great-circle + projected** | **✓ via PostGIS ecosystem** |

→ **wiki 内 native spatial vendor 从 1 增加到 2**: Vespa native + pgvector + PostGIS combo.

### pgvector + PostGIS 多模态 + spatial native pattern（NEW）

[per Postgres ecosystem + pgvector + PostGIS standard]

```sql
-- Multimodal + spatial + scalar + vector 同 schema
CREATE TABLE products (
  id bigserial PRIMARY KEY,
  description text,
  category text,
  price numeric,
  location geometry(Point, 4326),     -- PostGIS native R-tree
  embedding vector(768)                -- pgvector native HNSW
);

CREATE INDEX ON products USING gist (location);                          -- PostGIS spatial
CREATE INDEX ON products USING hnsw (embedding vector_cosine_ops);       -- pgvector HNSW
CREATE INDEX ON products (category);                                     -- B-tree filter
CREATE INDEX ON products USING gin (to_tsvector('english', description)); -- FTS

-- Query: "Italian leather wallet near me, <$500"
SELECT id, name, price,
       ST_Distance(location, ST_MakePoint($lng, $lat)::geography) AS dist_meters,
       embedding <=> $clip_embedding AS vec_dist
FROM products
WHERE ST_DWithin(location, ST_MakePoint($lng, $lat)::geography, 5000)  -- 5km bbox via PostGIS
  AND category = 'wallet'
  AND price < 500
ORDER BY embedding <=> $clip_embedding
LIMIT 10;
```

**4 modality 同 query 全部 native via Postgres**:
- Vector ANN (pgvector HNSW)
- Spatial filter (PostGIS R-tree)
- Scalar filter (B-tree)
- Full-text search (Postgres FTS)

→ pgvector + PostGIS **是 wiki 内第 2 个能 native 处理 4-modality query 的 path** (与 Vespa 并列).

### Vespa vs pgvector + PostGIS 4-modal benchmark candidate 对比

[per topics/multimodal-embedding-retrieval.md + Vespa + pgvector]

| | Vespa | pgvector + PostGIS |
|---|---|---|
| 4 modality (sparse + dense + multimodal + spatial) | ✓ first-class in single schema | ✓ via Postgres ecosystem |
| 整合方式 | tensor framework + ONNX inline CLIP + position dim + rank-profile | SQL standard + extension composition |
| Production 主流场景 | search engine / recommendation pipeline (Yahoo! lineage) | 已 deploying Postgres + AI features |
| Sharding architecture | content cluster groups + container | Postgres replication / Citus / app sharding |
| Scale ceiling | giga-scale production validated (Yahoo!) | ≤100M docs sweet spot |
| Operational complexity | dedicated infrastructure | reuse Postgres ops expertise |

→ Vespa 适合大规模 specialized, pgvector + PostGIS 适合**中等规模 + 已 Postgres ecosystem + 4-modality native**.

### Multimodal-spatial-hybrid benchmark methodology (updated 2026-05-12 post pgvector-docs)

**Tier 1: 4-modality native vendor inventory**:
- **Vespa**: ✓ 4 modality native first-class
- **pgvector + PostGIS** (NEW): ✓ 4 modality via Postgres ecosystem
- Others: 缺至少一个 (sparse / dense / multimodal / spatial)

**Tier 2: Fair benchmark protocol**:
- Vespa vs pgvector + PostGIS head-to-head 在 production multimodal query (e.g., visual search "near me"): 不公开
- Other vendor 通过 application-fanout: 需 report app overhead 单独

**Tier 3: Critical pitfall**:
- 忽略 pgvector + PostGIS 作 4-modality candidate: 之前 wiki 假设 native 仅 Vespa
- pgvector + PostGIS production case at scale 实际 deployment 不公开但 technically supported

### 已知盲区

- **pgvector + PostGIS 实际 production 4-modal case**: 不公开
- **Vespa vs pgvector + PostGIS 4-modal head-to-head**: 不公开
- **pgvector + PostGIS scale ceiling**: 假设 ≤100M docs (pgvector limit) 但 PostGIS 可独立 scale 不同
- **Wiki industry coverage 现状更新**: 之前 "仅 Vespa native spatial" assertion 需修正为 "Vespa + pgvector + PostGIS" 两个 native candidate

## Cited Pages

- [systems/pgvector.md](../../systems/pgvector.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [systems/vespa.md](../../systems/vespa.md)
