---
query-key: multimodal-bench-methodology
query: "对比多个 vector DB 在向量 + 标量 + 空间三模检索下的性能，怎么公平 benchmark？关键挑战：空间能力严重不对称（真索引 / bbox 近似 / 完全没有），怎么处理？"
date: 2026-05-12
phase: post
ingest-context: santhanam-2022-colbertv2
wiki-pages-total: 79
cited-pages: [concepts/colbertv2.md, topics/multimodal-embedding-retrieval.md]
cited-count: 2
---

# Post-snapshot (santhanam-2022-colbertv2): multimodal-bench-methodology

## TL;DR (post-ingest of santhanam-2022-colbertv2)

**ColBERTv2 是 text-only late-interaction**, 不涉及 multimodal embedding (CLIP-style cross-modal). 但 ColBERT-style late-interaction 可平移到 **multimodal late-interaction** (per token / per image patch / per modality 一 vector + MaxSim). **关键 NEW**: BGE-M3 已实现 single model 同时输出 sparse + dense + ColBERT-style late-interaction, 是 wiki 多模态 + 多 axis retrieval 的 unified future direction.

## Answer

### Multi-modal × late-interaction extension (theoretical)

[per santhanam-2022-colbertv2 + community work]

ColBERT-style MaxSim 可平移:
- Per image patch (ViT tokens): image-side late-interaction
- Per modality (image + text + audio multi-token): cross-modal late-interaction
- Per chunk in long-context: passage-level late-interaction

**Production extensions** (community-driven, paper-side 不深入):
- ColPali / ColCLIP: image-side late-interaction (实验性)
- Multi-vector vision-language models

### Wiki 9 vendor 空间/多模态/late-interaction matrix (NEW)

| Vendor | Native spatial | Cross-modal (CLIP) | Late-interaction (ColBERT) |
|---|---|---|---|
| Milvus | ✗ | partial 多 vector | **✓ 多 vector field native** |
| Qdrant | 半 native geohash | partial | partial multiple vectors per point |
| Weaviate | ✗ | partial | partial named vectors |
| Pinecone | 不公开 | ✓ Pinecone Inference | ✓ Multi-Vector Index (闭源) |
| **Vespa** | **✓ position** | **✓ tensor + ONNX CLIP** | **✓ tensor + weightedset native** |
| Turbopuffer | ✗ | partial | application 层 |
| pgvector | + PostGIS | partial | application 层 |
| Chroma | ✗ | partial | application 层 |
| LanceDB | ✗ | partial | Lance format multi-vector natural |

→ **Vespa 是 wiki 内唯一同时 native 4-axis** (spatial + multimodal + sparse + late-interaction) vendor.

### 已知盲区

- **ColPali / 视觉 late-interaction production case**: 不公开
- **Multi-modal MaxSim aggregation 公平 benchmark**: 不存在
- **BGE-M3 single-model 三 vector + multimodal extension**: 未明示

## Cited Pages

- [concepts/colbertv2.md](../../concepts/colbertv2.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
