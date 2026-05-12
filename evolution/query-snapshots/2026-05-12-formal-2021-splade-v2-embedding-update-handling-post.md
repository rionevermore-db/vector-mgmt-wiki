---
query-key: embedding-update-handling
query: "当 embedding model 升级（比如从 BERT 换到 SBERT，或 OpenAI embedding v2 → v3）时，已有的向量索引和原始文档应该怎么处理？是否必须全量重建？有没有跨模型兼容/复用的方案？"
date: 2026-05-12
phase: post
ingest-context: formal-2021-splade-v2
wiki-pages-total: 75
cited-pages: [concepts/splade-sparse-retrieval.md, concepts/clip.md, concepts/matryoshka-embedding.md, topics/sparse-dense-hybrid-retrieval.md]
cited-count: 4
---

# Post-snapshot (formal-2021-splade-v2): embedding-update-handling

## TL;DR (delta from kusupati-2022-matryoshka post)

**SPLADE 引入 wiki embedding upgrade 维度: sparse model 升级与 dense model 升级解耦**——dense model (CLIP/MRL) 升级影响 dense ANN index; SPLADE 升级影响 sparse inverted index. **两路独立**, hybrid retrieval pipeline 允许**逐路升级**: 可以仅升级 dense path (重 embed dense vector + 重建 HNSW), sparse path 不动; 或仅升级 sparse path (重 SPLADE encode + 重建 inverted index). 这给 production embedding upgrade workflow 提供 **降低风险的 incremental path**——之前 wiki 内仅 dense upgrade pattern, 现在双 path 独立可让 upgrade 风险分担.

## Answer

### 与之前 ingest 的演进

| | kusupati-2022-matryoshka post | **formal-2021-splade-v2 post (NEW)** |
|---|---|---|
| Upgrade path 数量 | 1 (dense) | **2 (dense + sparse) 独立可升级** |
| Within-model multi-tier | MRL prefix 0-cost | **不变 (dense path)** |
| Cross-model semantic preserve | 仍未关闭 | **不变** |
| Sparse path upgrade | 未涵盖 | **NEW: SPLADE retrain + inverted index 重建, 独立 dense path** |

### Hybrid pipeline 双 path 独立升级 (NEW)

[per formal-2021-splade-v2 + Phase 1-2 + production reality]

**Pre-SPLADE upgrade workflow (dense only)**:
- 升级 BERT-base → OpenAI text-embedding-3 → 重 embed corpus + 重建 HNSW
- 单 path, single point of risk

**Post-SPLADE upgrade workflow (hybrid pipeline)**:
- Dense path: voyage-3 → voyage-4 → 重 embed dense + 重建 HNSW
- Sparse path: SPLADE-v2 → SPLADE-v3 → 重 SPLADE encode + 重建 inverted index
- **两 path 独立升级**, partial-failure recovery:
  - 升级 dense 失败 → fall back to sparse only (still functional, 略 worse recall)
  - 升级 sparse 失败 → fall back to dense only (still functional)
  - 两者同时升级 → 完整 upgrade

### Production hybrid pipeline upgrade pattern

[per topics/sparse-dense-hybrid-retrieval.md + 5 vendor]

| Vendor | Sparse upgrade path | Dense upgrade path | Hybrid coordination |
|---|---|---|---|
| **Vespa** | rank-profile sparse field 重 index | tensor field 重 index | application package atomic deploy 同步 |
| **Weaviate** | BlockMaxWAND 重 index (BM25 自动) | HNSW 重 index | `hybrid()` API alpha 仍 work |
| **Pinecone** | Sparse Index 重 upsert | Dense Index 重 upsert | Hybrid Index 双 index 同步 |
| **Turbopuffer** | namespace BM25 重 index | namespace dense 重 index | multi_query 客户端 fusion 仍 work |
| **Milvus** | sparse vector field 重 build | dense vector field 重 build | multi-field collection upgrade |

→ **5 vendor 都支持双 path 独立升级**——hybrid 架构天然提供 upgrade isolation.

### Algorithm 层 cross-model 仍未关闭

[per formal-2021-splade-v2 §6 + wiki frontier]

SPLADE 同样**不解决** algorithm 层 cross-model semantic preserve:
- SPLADE-v2 → SPLADE-v3 cross-model embedding 几何不一定 compatible (vocab 大致同 但 term weight distribution 不同)
- SPLADE → DeepImpact / COIL 跨模型也 不 compatible
- **跨 sparse algorithm migration** 仍需重 encode

→ **Algorithm 层跨模型 preserve frontier 仍未关闭** (无论 sparse 还是 dense). Talk SIGMOD 2026 live demo 仍是唯一 candidate.

### 处理决策表（updated 2026-05-12 post formal-2021-splade-v2）

| 场景 | 推荐方案 |
|---|---|
| **Hybrid pipeline 一次仅升级一路 (sparse OR dense)** | **5 vendor 全支持; 降低 upgrade risk 标准 pattern** |
| Within-MRL-model multi-tier (mobile/cloud/archival) | 单 schema + 应用层 prefix dim 选择 (0 re-embed) |
| Dense upgrade major (voyage-3 → voyage-4 dim change) | Vespa multi-tensor / Turbopuffer namespace per version |
| **Sparse upgrade SPLADE-v2 → v3** | 重 encode + 重建 inverted index; 独立 dense path |
| Cross-model semantic preserve | 仍未有 production——SIGMOD 2026 |

### 已知盲区

- **SPLADE-v2 vs v3 / SPLADE++ embedding compatibility**: 论文不验证, production 不公开
- **Hybrid 一路升级时 fusion α / RRF k 是否需要 retrain**: 不公开
- **5 vendor 客户实际 hybrid upgrade workflow case study**: 不公开
- **Sparse algorithm cross-model**: SPLADE → DeepImpact / BGE-M3 sparse 路径 compatible? Open

## Cited Pages

- [concepts/splade-sparse-retrieval.md](../../concepts/splade-sparse-retrieval.md)
- [concepts/clip.md](../../concepts/clip.md)
- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
- [topics/sparse-dense-hybrid-retrieval.md](../../topics/sparse-dense-hybrid-retrieval.md)
