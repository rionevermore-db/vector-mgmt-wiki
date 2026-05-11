---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-11
phase: post
ingest-context: radford-2021-clip
wiki-pages-total: 71
cited-pages: [concepts/hnsw.md, concepts/nsg.md, concepts/clip.md, topics/multimodal-embedding-retrieval.md]
cited-count: 4
---

# Post-snapshot (radford-2021-clip): hnsw-vs-nsg-selection

## TL;DR (delta from adams-2025-distributedann post)

**CLIP ingest 不改变 HNSW vs NSG 选择决策**——CLIP 是 embedding model layer 算法, 与 ANN index 算法 (HNSW/NSG/Vamana) 是**正交关系**: CLIP 决定 vector 是什么 (768-d L2-normalized cosine-friendly multimodal embedding), HNSW/NSG 决定如何 index 这些 vector. **关键 NEW**: HNSW 在 wiki 内 **multimodal production workload 默认 ANN 选择**——4 OSS vector DBMS (Milvus/Qdrant/Weaviate/Vespa) + 1 closed SaaS (Pinecone) + 1 commercial SaaS (Turbopuffer SPFresh path) 服务 CLIP-style multimodal retrieval, **HNSW 是其中 5/7 的主要 ANN backbone**——NSG production multimodal case 仍零.

## Answer

### 与之前 ingest 的演进

| | adams-2025-distributedann post | **radford-2021-clip post (NEW)** |
|---|---|---|
| HNSW production OSS DBMS | 4 (Milvus + Qdrant + Weaviate + Vespa) | **不变** (CLIP 不影响 ANN index 选择) |
| NSG production | Taobao 2B + NSSG | **不变** (no multimodal NSG production case) |
| HNSW serving CLIP-style multimodal embedding production | wiki 未明示 | **NEW: HNSW 是 wiki 5/7 vendor 的 multimodal default ANN** |
| Distance metric for CLIP | n/a (未涵盖) | **cosine / inner product (CLIP L2-normalized embedding 上等价)** |

### CLIP 引入的 normalization assumption（NEW）

[per radford-2021-clip §2.4 Figure 3]

```python
I_e = l2_normalize(np.dot(I_f, W_i), axis=1)  # image embedding
T_e = l2_normalize(np.dot(T_f, W_t), axis=1)  # text embedding
```

→ CLIP embeddings **总是 L2-normalized** (output 在 unit sphere 上). 因此 cosine similarity = inner product. **所有 wiki vector DBMS 都 first-class 支持 cosine ANN**, HNSW 不需要任何 modification 即服务 CLIP retrieval workload.

### HNSW vs NSG 在 multimodal production workload

| 工程考量 | HNSW (5/7 vendor multimodal default) | NSG (no multimodal production) |
|---|---|---|
| Cosine ANN over L2-normalized CLIP | **trivial** (默认距离 metric) | trivial 但 production case 零 |
| Filter-aware multimodal (CLIP + category filter) | **ACORN (Qdrant + Weaviate + Vespa) + Filterable HNSW** | ✗ |
| Multi-CLIP-version coexistence | **Turbopuffer namespace + Vespa multi-tensor field + Weaviate named vectors** all HNSW base | ✗ |
| CLIP image + text dual vector | **Milvus multi-vector field + Vespa tensor framework** all HNSW base | ✗ |
| OPQ / RaBitQ / Binary quantization on CLIP | Qdrant BQ + Weaviate BQ + Vespa cell type all HNSW base | ✗ |

### Multimodal production workload 选择决策（NEW）

| Workload | 推荐 |
|---|---|
| **Multimodal retrieval + tensor framework + 4-phase ranking** | **Vespa HNSW** (CLIP inline encoder + multi-tensor field) |
| Multimodal + 多 index_type | Milvus HNSW / DISKANN / CAGRA path |
| Multimodal + OSS Rust + 简洁 | Qdrant HNSW |
| Multimodal + AI-native primary DB + agent stack | Weaviate HNSW + named vectors |
| Multimodal + 闭源 SaaS managed | Pinecone HNSW-implied |
| Multimodal + per-tenant 独立 model | Turbopuffer namespace-as-tenant + SPFresh path |

→ **HNSW 5 OSS DBMS 默认 multimodal ANN backbone**——NSG 在 multimodal 仍零 production.

### 已知盲区

- **NSG 服务 CLIP-style 实测**: 学术 benchmark 都用 SIFT/GIST 不是 CLIP embedding——NSG vs HNSW on real CLIP 768-d 实测?
- **CLIP embedding distribution 对 HNSW α-RNG property 的影响**: CLIP 训练后 distribution shift 与 HNSW 静态 graph 假设的 mismatch?
- **HNSW + Vamana 多版本 CLIP migration**: ViT-B/32 (512-d) → ViT-L/14 (768-d) embedding model 升级时 graph 重建成本
- **DistributedANN-style single graph on CLIP**: Bing DistributedANN paper 不公开是否服务 multimodal——50B Vamana graph on multi-modal embedding 实证未知

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/nsg.md](../../concepts/nsg.md)
- [concepts/clip.md](../../concepts/clip.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
