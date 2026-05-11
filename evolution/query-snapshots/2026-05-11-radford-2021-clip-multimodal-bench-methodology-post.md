---
query-key: multimodal-bench-methodology
query: "对比多个 vector DB 在向量 + 标量 + 空间三模检索下的性能，怎么公平 benchmark？关键挑战：空间能力严重不对称（真索引 / bbox 近似 / 完全没有），怎么处理？"
date: 2026-05-11
phase: post
ingest-context: radford-2021-clip
wiki-pages-total: 71
cited-pages: [concepts/clip.md, topics/multimodal-embedding-retrieval.md, systems/vespa.md, topics/attribute-filtering.md]
cited-count: 4
---

# Post-snapshot (radford-2021-clip): multimodal-bench-methodology

## TL;DR (post-ingest of radford-2021-clip)

**重要语义澄清**: query 用"多模态" (multimodal) 一词指 "vector + scalar + spatial"——但 CLIP ingest 后 wiki 有 **第二种 multimodal 定义**: cross-modal embedding (image+text+audio+depth). 两种 multimodal 含义对应**两个不同的 wiki industry coverage gap**:
- **Tri-modal query (vector + scalar + spatial)**: 6 vendor 仅 Vespa native spatial; **fragmented capability landscape**
- **Cross-modal embedding (CLIP family)**: 6 vendor cosine ANN 全 transparent service; **algorithm 成熟, benchmark zero coverage**

**关键 NEW**: 把两种 multimodal 组合 = "tri-modal + cross-modal multimodal" = e.g., CLIP image embedding + brand filter + spatial bbox filter——是 **production 实际 workload**, e.g., visual search "show me Italian leather wallets near me ≤5km"——**完全 wiki industry coverage gap**.

## Answer

### Wiki 7 vendor 空间能力 inventory（unchanged from turbopuffer-docs post）

| Vendor | 真 spatial 路径 |
|---|---|
| Milvus | ✗ |
| Qdrant | 半 native (geohash filter) |
| Weaviate | ✗ |
| Pinecone | 不公开 |
| **Vespa** | **✓ native** (position dimension) |
| Turbopuffer | ✗ |
| DistributedANN | paper 不涉及 |

### Multimodal 语义两种含义（NEW critical clarification）

[per radford-2021-clip + topics/multimodal-embedding-retrieval.md]

**含义 1: Tri-modal query (Query #9 原意)**:
- Vector + scalar filter + spatial filter (lat/lng / bbox)
- 6 vendor 仅 Vespa native spatial
- **Algorithm 分裂**: native R-tree / geohash / bbox-as-attribute / 无

**含义 2: Cross-modal embedding (CLIP / SigLIP / ImageBind)**:
- Multimodal embedding model (image+text+audio+...) → joint embedding space → cosine ANN
- 6 vendor 全 transparent service (cosine ANN)
- **Algorithm 成熟**: CLIP family is industry standard
- **Benchmark 空白**: 不存在公开 vendor-vs-vendor multimodal head-to-head

### Multimodal 综合 query 实际 production case（NEW）

[per real e-commerce / RAG / visual search workload]

实际 production multimodal query 常 **两种含义同时存在**:
1. **Visual search nearby**: 上传图片 (CLIP image embed) + bbox filter "5km within home" + category="furniture" + price<500
2. **Local RAG**: text query (CLIP text embed) + spatial filter "city=Boston" + source="local-docs"
3. **AI shopping assistant**: text + image attached (CLIP image embed) + brand + spatial preference + price range

→ 这类 query = (cross-modal CLIP embedding) + (scalar filter) + (spatial filter) = **真 tri-modal + cross-modal**.

### 公平 benchmark 框架（updated 2026-05-11 post radford-2021-clip）

[per multimodal embedding + topics/attribute-filtering.md + spatial reality]

**Tier 1: Capability inventory 矩阵**:

| Vendor | Cross-modal embedding service | Native spatial index | Multi-modal hybrid query API |
|---|---|---|---|
| Vespa | ✓ tensor + ONNX inline CLIP | ✓ native position dim | **✓ (only)** |
| Milvus | ✓ multi-vector field | ✗ | filter + vector |
| Qdrant | ✓ multiple vectors per point | 半 native geohash | filter + geohash + vector |
| Weaviate | ✓ named vectors | ✗ | filter + vector |
| Pinecone | ✓ Pinecone Inference CLIP-style | 不公开 | filter + vector (黑盒) |
| Turbopuffer | ✓ namespace per CLIP variant | ✗ | filter + vector |
| DistributedANN | paper 不涉及 | paper 不涉及 | paper 不涉及 |

**Tier 2: Workload 设计 (公平 benchmark)**:
1. **Cross-modal-only**: CLIP image embedding + simple filter (没 spatial)
2. **Spatial-only**: vector + spatial filter (Vespa native vs 其他 application fanout)
3. **Tri-modal + cross-modal**: CLIP embedding + scalar filter + spatial filter (全 vendor 缺 path)

**Tier 3: 实测协议**:
- Cross-modal Recall@10 / Recall@200 measured against exhaustive (CLIP embedding requires exhaustive baseline)
- Spatial filter behavior: vendor-specific (geohash precision / bbox approximate)
- Multi-modal fusion: application-layer vs vendor-native

**Tier 4: Pitfall**:
- 用 SIFT 替代 CLIP embedding——错 (distribution 不同)
- 仅测 dense vector 单 modality——错 (production 是 hybrid)
- 假设全 vendor 相同 spatial 能力——错 (Vespa native + Qdrant 半 native + 其他 fanout)

### Multimodal embedding × spatial × scalar 真 tri-modal frontier

[per wiki industry coverage]

**当前 wiki 完全空白**:
- CLIP-embedded production case + spatial filter (Vespa) head-to-head 不存在
- E-commerce visual search "nearby + category + image similar" 实测 benchmark 不存在
- RAG with local context (text CLIP + spatial filter) 不存在公开 case

→ **这是 wiki 内 most important production workload coverage gap**——CLIP multimodal 算法成熟 + spatial 在 Vespa 成熟, 但 **两者交叉的 production case 完全缺失**.

### 已知盲区（updated 2026-05-11 post radford-2021-clip）

- **Cross-modal + spatial + scalar tri-modal benchmark**: wiki + vendor 完全空白
- **Vespa CLIP + position + filter 实测**: vendor 在 multimodal frontier 唯一 candidate 但 production case 不公开
- **多 CLIP variant + spatial 同 schema**: production routing 不公开
- **Vision search "nearby" 与 普通 multimodal retrieval cost 差异**: 不公开

## Cited Pages

- [concepts/clip.md](../../concepts/clip.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [systems/vespa.md](../../systems/vespa.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
