---
query-key: hnsw-vs-nsg-selection
query: "HNSW 和 NSG 都是基于 proximity graph 的 ANN 算法。从工程角度（构建成本、查询性能、动态更新支持、内存开销），它们分别适合什么场景？"
date: 2026-05-11
phase: post
ingest-context: kusupati-2022-matryoshka
wiki-pages-total: 73
cited-pages: [concepts/hnsw.md, concepts/matryoshka-embedding.md, topics/adaptive-retrieval-shortlist-rerank.md]
cited-count: 3
---

# Post-snapshot (kusupati-2022-matryoshka): hnsw-vs-nsg-selection

## TL;DR (delta from radford-2021-clip post)

**MRL ingest 不改变 HNSW vs NSG 算法选择**——MRL 是 embedding-side training-time technique, 与 ANN index 算法 (HNSW/NSG/Vamana) 是**正交关系**. **关键 NEW**: 但 MRL 给 HNSW 引入一个**新 production deployment pattern**——**multi-prefix-dim HNSW indexing**: 同 corpus 可建 (a) 16-d prefix HNSW (cheap shortlist) + (b) 1024-d full HNSW (rerank), 或单 1024-d HNSW + 应用层 prefix query. 这是 **HNSW vs NSG 选择决策中, HNSW 因生态成熟优势进一步扩大**——NSG 在 MRL ecosystem 中 zero production deployment. paper §6 future work 提 "MRL-aware ANN index (learnable k-d tree)"—— 但目前所有 vendor 用 generic HNSW + 应用层/native MRL truncation.

## Answer

### 与之前 ingest 的演进

| | radford-2021-clip post | **kusupati-2022-matryoshka post (NEW)** |
|---|---|---|
| HNSW production OSS DBMS | 4 (Milvus + Qdrant + Weaviate + Vespa) | **不变** |
| NSG production multimodal | Taobao 2B only | **不变** |
| HNSW + MRL prefix integration | n/a | **NEW: production state-of-the-art = HNSW + MRL prefix truncation + Adaptive Retrieval** |
| MRL-aware ANN index | paper §6 future work | **不存在 production** |

### HNSW + MRL production pattern（NEW）

[per kusupati-2022-matryoshka §4.3 + wiki vendor]

**Pattern A: single HNSW + prefix query (默认 production)**:
- vector DB 存 full d-维 (e.g., 1024-d voyage-3) MRL-trained embedding
- HNSW index 建在 full dim 上
- query 端选 prefix dim → 应用层先 truncate query embedding, 再做 ANN
- 所有 5 OSS DBMS + Pinecone + Turbopuffer 都 transparent 支持

**Pattern B: multi-prefix HNSW indexing (Vespa matryoshka cell type native)**:
- 单 doc 存 1024-d full + 同时建 256-d prefix HNSW + 1024-d full HNSW
- shortlist 用 256-d prefix HNSW; rerank 用 1024-d full HNSW (或全维 exact)
- **Vespa 唯一 native first-class 支持**

**Pattern C: Funnel HNSW cascade (paper §4.3.1)**:
- 多 stage shortlist 16→32→64→128→256→2048
- 每 stage 一个 HNSW index (或共享 full HNSW + 不同 prefix query)
- production case zero (paper-level only)

### NSG production zero coverage in MRL ecosystem

[per wiki + kusupati-2022-matryoshka]

NSG 在 MRL-trained embedding (text-embedding-3 / voyage-3 / embed-v4 等) 上 zero production case. **原因推测**:
- MRL-trained embedding 通常 1024-d+ (vs SIFT 128-d), NSG 单 entry point + 静态 graph 在高维 cosine 几何上**经验上 worse than HNSW**
- HNSW 多 OSS DBMS 集成成熟; NSG 仅 academic baseline
- 没有 vendor 实现 NSG + MRL combination

### 选择决策（updated 2026-05-11 post kusupati-2022-matryoshka）

- **MRL-trained embedding production + multi-stage AR pipeline** → **Vespa HNSW + matryoshka cell type + 4-phase ranking** (唯一 native Pattern B)
- MRL-trained embedding + OSS vector DBMS + application-level AR → HNSW on any of Milvus / Qdrant / Weaviate / Turbopuffer / Pinecone
- 非-MRL embedding (legacy SIFT/GIST/CLIP-base) + production → HNSW (4 OSS vendor)
- CPU + single-vector TopK + 静态 SIFT-style → NSG (学术 benchmark only)

### 已知盲区

- **MRL prefix 上 HNSW α-RNG property 退化**: 1024-d HNSW α-RNG 在 256-d prefix 上是否仍 work? 未量化
- **MRL-aware ANN index** (paper §6 future): 显式利用 nested structure 的 ANN index 设计未做
- **NSG + MRL**: 完全 unknown territory——NSG 单 entry 在 nested prefix 上的行为
- **Vespa matryoshka cell type 实际 production scale**: docs 提及但 case study zero

## Cited Pages

- [concepts/hnsw.md](../../concepts/hnsw.md)
- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
- [topics/adaptive-retrieval-shortlist-rerank.md](../../topics/adaptive-retrieval-shortlist-rerank.md)
