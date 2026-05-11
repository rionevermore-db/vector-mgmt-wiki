---
query-key: vector-scalar-bench-methodology
query: "对比多个 vector DB 在向量 + 标量过滤混合查询下的性能，如何公平地横向 benchmark？"
date: 2026-05-11
phase: post
ingest-context: radford-2021-clip
wiki-pages-total: 71
cited-pages: [concepts/clip.md, topics/multimodal-embedding-retrieval.md, topics/attribute-filtering.md]
cited-count: 3
---

# Post-snapshot (radford-2021-clip): vector-scalar-bench-methodology

## TL;DR (post-ingest of radford-2021-clip)

**CLIP ingest 不直接改变 vector-scalar bench methodology**——CLIP 提供 vector content (multimodal embedding), 与 scalar filter 是正交维度. **关键 NEW**: 但 CLIP-style multimodal embedding 引入**新 benchmark workload reality**——production vector-scalar 查询大多是 **multimodal context**: "text query CLIP-embedded + brand filter + category filter", 不是简单 "text embedding + brand filter". **公平 benchmark 需 reflect 这一现实**: dataset 选 production-realistic multimodal queries (e.g., MS-COCO + filter / Flickr30K + caption metadata), 而非纯 SIFT/GIST + 人工 filter.

## Answer

### Wiki 当前 vendor filter 实现表（unchanged from turbopuffer-docs post）

7 vendor filter 实现 (Milvus / Qdrant / Weaviate / Vespa / Pinecone / Turbopuffer / DistributedANN-implied) 不变.

### CLIP-aware vector-scalar benchmark methodology（NEW）

[per radford-2021-clip multimodal reality + topics/attribute-filtering.md]

**Production vector-scalar workload 典型 case**:
- E-commerce: "Italian leather wallet" CLIP embed + category="wallet" + brand="X" + price<100
- RAG: technical question CLIP embed + source="docs" + last_modified>2026
- Visual search: query image CLIP embed + category filter

**当前 wiki vector-scalar benchmark gap**:
- 没有 vendor 公开 CLIP-style multimodal workload + filter 实测
- ANN benchmark 标准 dataset (SIFT/GIST/DEEP/BIGANN) 都不含 multimodal context
- LAION-CLIP / WIT-CLIP embeddings 在 ANN benchmark 上的 filter behavior **完全未知**

**公平 benchmark 框架 (updated 2026-05-11 post radford-2021-clip)**:

1. **Dataset diversity**: 同时测试
   - 经典: SIFT/GIST/DEEP/BIGANN (text-style / 视觉特征, ANN benchmark 主流)
   - **NEW: CLIP-embedded**: LAION-400M / MS-COCO / Conceptual Captions 等 multimodal embedding 实际 distribution
   - 真实 workload: e-commerce product embedding (CLIP fine-tune for product, 不公开)

2. **Embedding distribution awareness**: 
   - SIFT embedding 几何与 CLIP 几何不同 (CLIP L2-normalized + joint multimodal anchored)
   - Filter selectivity 在不同 embedding distribution 下表现可能不同 (Weaviate "correlation optimization" 在 CLIP 上是否同 effective?)
   - 必须报 embedding type + dataset + recall threshold

3. **Cross-modal filter case**:
   - text query CLIP + image-side metadata filter (image 拍摄 country=US)
   - image query CLIP + text-side metadata filter (caption 含 "limited edition")
   - 这种 **cross-modal filter** 在 wiki zero coverage

4. **Production-realistic selectivity**:
   - 之前框架: selectivity ∈ {0.01%, 0.1%, 1%, 10%, 50%, 90%, 99%}
   - **NEW: 加入 modality-aware selectivity**: "filter pre-selects 100% docs of one modality (e.g., only images), filter selectivity is meaningful only within remaining set"

### Multimodal-aware vector-scalar benchmark pitfall（NEW）

[per CLIP embedding + production reality]

1. **不区分 embedding distribution**: 用 SIFT result generalize 到 CLIP——错
2. **Pure text-side benchmark**: 仅测 BGE / SBERT, 不测 CLIP——production 实际是 multimodal
3. **静态 corpus**: production CLIP corpus 随 model 升级动态变, benchmark 应反映 model migration cost
4. **忽略 cross-modal asymmetry**: text→image 与 image→image 在 quantization / filter / re-rank 上行为可能不同

### 已知盲区（updated 2026-05-11 post radford-2021-clip）

- **CLIP-embedded ANN benchmark dataset**: 不存在公开标准
- **Multimodal vector + scalar filter head-to-head**: 7 vendor 全 zero coverage
- **CLIP embedding distribution shift 后 filter behavior**: 不存在公开数据
- **Cross-modal filter (text query + image metadata, or vice versa) benchmark**: wiki zero coverage

## Cited Pages

- [concepts/clip.md](../../concepts/clip.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [topics/attribute-filtering.md](../../topics/attribute-filtering.md)
