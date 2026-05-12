---
query-key: multimodal-bench-methodology
query: "对比多个 vector DB 在向量 + 标量 + 空间三模检索下的性能，怎么公平 benchmark？关键挑战：空间能力严重不对称（真索引 / bbox 近似 / 完全没有），怎么处理？"
date: 2026-05-12
phase: post
ingest-context: formal-2021-splade-v2
wiki-pages-total: 75
cited-pages: [concepts/splade-sparse-retrieval.md, topics/sparse-dense-hybrid-retrieval.md, topics/multimodal-embedding-retrieval.md, systems/vespa.md]
cited-count: 4
---

# Post-snapshot (formal-2021-splade-v2): multimodal-bench-methodology

## TL;DR (post-ingest of formal-2021-splade-v2)

**SPLADE 给 multimodal 维度加 "sparse text retrieval is text-only" 限制——multimodal embedding (CLIP/SigLIP/ImageBind) 都是 dense, SPLADE 是 text sparse**. **关键 NEW**: production tri-modal benchmark 必须区分 (a) text-side sparse (SPLADE / BM25), (b) text-side dense (BGE / Voyage), (c) cross-modal dense (CLIP / SigLIP), (d) spatial filter (Vespa native / Qdrant geohash / 其他 attribute-only). **Vespa 是 wiki 内同时 first-class native** sparse (BM25 + SPLADE weightedset) + dense (tensor framework) + multimodal (ONNX inline CLIP encoder) + spatial (position dimension) **4 axis 唯一 vendor**——production multimodal-spatial-hybrid benchmark 候选.

## Answer

### Wiki 7 vendor 4-modality 能力 inventory (NEW comprehensive)

[per topics/sparse-dense-hybrid-retrieval.md + topics/multimodal-embedding-retrieval.md + formal-2021-splade-v2]

| Vendor | Sparse path | Dense path | Multimodal path (image/CLIP) | Spatial path |
|---|---|---|---|---|
| **Vespa** | **✓ BM25 + SPLADE weightedset first-class** | **✓ tensor framework** | **✓ ONNX inline CLIP encoder** | **✓ position dimension** |
| Milvus | ✓ sparse_inverted_index | ✓ 多 vector field | ✓ 多 vector field | ✗ |
| Qdrant | ✓ sparse vector (v1.7+) | ✓ HNSW + 多 vector | ✓ 多 vector | 半 native (geohash filter) |
| Weaviate | ✓ BlockMaxWAND BM25 | ✓ HNSW + RQ8 | ✓ named vectors | ✗ |
| Pinecone | ✓ Sparse Vector Index | ✓ Dense Vector Index | ✓ Pinecone Inference | 不公开 |
| Turbopuffer | ✓ BM25 (BlockMaxWAND-style) | ✓ SPFresh + cosine | ✓ namespace per CLIP variant | ✗ |
| DistributedANN | paper 不涉及 | distributed KV graph | paper 不涉及 | paper 不涉及 |

→ **Vespa 同时 first-class native 4 axis 唯一 vendor** — 是 production tri/quad-modal-hybrid benchmark 候选.

### Multimodal-spatial-hybrid public benchmark methodology (updated)

[per formal-2021-splade-v2 + Phase 1-2 + topics/multimodal-embedding-retrieval.md]

**Tier 1: Capability matrix 4 axis**:
- **Sparse axis**: BM25 / SPLADE / DeepImpact / COIL
- **Dense axis**: CLIP / MRL / Voyage / Cohere / BGE
- **Multimodal axis**: text / image / cross-modal
- **Spatial axis**: bbox / lat-lng / true spatial (Vespa native)

**Tier 2: Workload (production-realistic)**:
- E-commerce visual search "near me": CLIP image + brand filter + spatial bbox
- Local RAG: text CLIP + city filter + source filter + recency filter
- Multimodal hybrid: text CLIP + product image + price + category + spatial

**Tier 3: Pitfall**:
- 仅 dense benchmark → 忽略 sparse retrieval 在 long-tail 的优势
- 仅 sparse benchmark → 忽略 multimodal semantic
- 仅 multimodal → 忽略 sparse exact-match
- 仅 spatial → 不反映 production real workload

### 4 axis benchmark wiki industry coverage gap (worst)

[per formal-2021-splade-v2 + Phase 1-2 + Vespa native]

**Wiki 内** ≥10B 4-axis (sparse + dense + multimodal + spatial) production case **完全空白**:
- Vespa technically capable 4 axis, 但无公开 production case
- 其他 vendor 至少缺一个 axis
- 论文层 SPLADE focus on text retrieval; CLIP focus on multimodal; 各自无 benchmark cover 4 axis

→ **这是 wiki 内 most important production workload coverage gap**——SPLADE + CLIP + spatial Vespa native 联合实际 production case 不存在公开.

### 已知盲区（updated 2026-05-12 post formal-2021-splade-v2）

- **4 axis production case (sparse + dense + multimodal + spatial)**: 完全空白
- **Sparse path 的 multimodal extension**: SPLADE text-only; sparse multimodal (image-side sparse retrieval) production case zero
- **BM25 vs SPLADE in hybrid benchmark on multimodal context**: paper-level SPLADE evaluate text-only, multimodal context Migration 不公开
- **Vespa 4 axis 实际 production scale**: docs 提及但 case study zero

## Cited Pages

- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [systems/vespa.md](../../systems/vespa.md)
