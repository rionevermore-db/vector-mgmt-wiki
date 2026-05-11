---
query-key: multimodal-bench-methodology
query: "对比多个 vector DB 在向量 + 标量 + 空间三模检索下的性能，怎么公平 benchmark？关键挑战：空间能力严重不对称（真索引 / bbox 近似 / 完全没有），怎么处理？"
date: 2026-05-11
phase: post
ingest-context: kusupati-2022-matryoshka
wiki-pages-total: 73
cited-pages: [concepts/matryoshka-embedding.md, concepts/clip.md, topics/multimodal-embedding-retrieval.md, topics/adaptive-retrieval-shortlist-rerank.md, systems/vespa.md]
cited-count: 5
---

# Post-snapshot (kusupati-2022-matryoshka): multimodal-bench-methodology

## TL;DR (post-ingest of kusupati-2022-matryoshka)

**MRL ingest 给 multimodal-bench-methodology 加 cross-modal-MRL 维度**——MRL 不仅是 text-only 优势, **ALIGN model (CLIP-like, vision+language) 也被 MRL retrained** (per kusupati-2022 §4.1)——可直接产出 cross-modal MRL embedding. **关键 NEW**: production multimodal embedding (Cohere multilingual / Voyage multimodal / Qwen3-VL-Embedding) **全部 MRL-trained**. Multimodal benchmark methodology 必须 sweep prefix dim, vendor-native MRL support 区分 Vespa matryoshka cell type vs 其他 application-level. 与 spatial 维度的 wiki industry gap (仅 Vespa native) 形成**双 frontier 对称**: multimodal MRL native + spatial native 都仅 Vespa.

## Answer

### 与之前 ingest 的演进

| | radford-2021-clip post | **kusupati-2022-matryoshka post (NEW)** |
|---|---|---|
| 7 vendor 空间能力 inventory | Vespa native only | **不变** |
| Multimodal benchmark gap | head-to-head zero | **+ MRL prefix dim 必须作为 benchmark axis** |
| Cross-modal MRL production case | n/a | **NEW: ALIGN-MRL + Voyage multimodal MRL + Qwen3-VL-Embedding** |
| Native multimodal + MRL + spatial | Vespa only (3 frontier 唯一交集) | **不变** — Vespa 唯一 |

### Cross-modal MRL: ALIGN-MRL + production case（NEW）

[per kusupati-2022-matryoshka §4.1 + production reality]

paper §4.1 明示 MRL 在 ALIGN model (CLIP-like vision+language) 上 retrain, 结果**与 ViT-B/16 vision-only MRL 一致**:
- Figure 4 (ImageNet-1K 1-NN): ALIGN MRL 在 12/24/48/96/192/384/768 dim 上**全部** 优于 fixed-feature baseline
- Cosine similarity span: ALIGN-MRL 改善 positive vs random image-text pairs 区分
- 适合 cross-modal retrieval 的 unique benefit: shortlist with prefix → rerank with full

**Production multimodal MRL embedding**:
- Voyage voyage-multimodal-3 (MRL + QAT trained)
- Cohere multilingual embed-v4 (MRL trained)
- Qwen3-VL-Embedding-8B (MRL trained, Turbopuffer docs 推荐)
- Snowflake / Mixedbread multimodal MRL variants

### Multimodal + spatial benchmark methodology (updated 2026-05-11 post kusupati-2022-matryoshka)

**Tri-modal + MRL 4 axis matrix**:
- **Axis 1: Modality** — text only / image only / text+image cross-modal
- **Axis 2: Spatial** — none / lat-lng filter / bbox / true spatial (Vespa position)
- **Axis 3: MRL prefix dim** — 64 / 128 / 256 / 512 / 768 / 1024
- **Axis 4: Filter selectivity** — 0.01% / 0.1% / 1% / 10% / 50%

**Tier 1: Capability matrix (cross-modal MRL × spatial × native)**:

| Vendor | Cross-modal MRL native | Spatial native | 4-axis benchmark candidate |
|---|---|---|---|
| **Vespa** | ✓ matryoshka cell type + ONNX inline CLIP | ✓ position dimension | **✓ (唯一)** |
| Milvus | application-level MRL + 多 vector field | ✗ (filter only) | partial (no spatial) |
| Qdrant | application-level MRL + 多 vector | 半 native geohash | partial (semi spatial) |
| Weaviate | application-level MRL + named vectors | ✗ | partial (no spatial) |
| Pinecone | 不公开 MRL support; Pinecone Inference CLIP-style | 不公开 | unknown |
| Turbopuffer | application-level MRL + namespace per QAT model | ✗ | partial (no spatial) |
| DistributedANN | paper 不明示 multimodal MRL | paper 不涉及 | partial (Bing-internal only) |

### Vespa 是 wiki 内唯一 3 frontier 同时 native 候选

[per kusupati-2022-matryoshka + radford-2021-clip + wiki industry coverage]

- **Spatial native**: position dimension (wiki 唯一)
- **Multimodal native**: tensor framework + ONNX inline CLIP encoder (wiki 唯一)
- **MRL native**: matryoshka cell type (wiki 唯一)
- **同时三者**: 仅 Vespa

→ **Vespa 是 wiki 内 Tri-modal + MRL + Spatial 同时 native first-class 唯一系统**, production benchmark candidate.

### 已知盲区

- **Cross-modal MRL × spatial 三 frontier 实测**: 仅 Vespa technically capable, production case zero
- **Multimodal MRL vendor benchmark**: 不存在公开
- **Cross-modal MRL prefix recall asymmetry**: text→image 与 image→text 在不同 prefix dim 下 recall 是否对称? Paper 不验证
- **多 vendor multimodal MRL fair benchmark**: 不存在

## Cited Pages

- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
- [concepts/clip.md](../../concepts/clip.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [topics/adaptive-retrieval-shortlist-rerank.md](../../topics/adaptive-retrieval-shortlist-rerank.md)
- [systems/vespa.md](../../systems/vespa.md)
