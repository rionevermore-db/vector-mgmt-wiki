---
query-key: embedding-update-handling
query: "当 embedding model 升级（比如从 BERT 换到 SBERT，或 OpenAI embedding v2 → v3）时，已有的向量索引和原始文档应该怎么处理？是否必须全量重建？有没有跨模型兼容/复用的方案？"
date: 2026-05-11
phase: post
ingest-context: kusupati-2022-matryoshka
wiki-pages-total: 73
cited-pages: [concepts/matryoshka-embedding.md, concepts/clip.md, topics/adaptive-retrieval-shortlist-rerank.md, systems/vespa.md, systems/turbopuffer.md]
cited-count: 5
---

# Post-snapshot (kusupati-2022-matryoshka): embedding-update-handling

## TL;DR (delta from radford-2021-clip post)

**MRL ingest 给 embedding upgrade frontier 带来 wiki 内最重要的工程层改进**——之前 wiki 内多 model variant 升级 (BERT-base→BERT-large, ViT-B/32→ViT-L/14) **必须**重 embed 全 corpus + 重建 vector DB index 因为 dim 不同 (512→768). **MRL 单一 model 多 prefix dim 让"多 variant"取代为"单 model 多 prefix dim 选择"**: 训练时一次 voyage-3 (1024-d), 部署后 mobile 取 256-d / cloud 取 1024-d / archival 取 512-d——**同一 vector DB schema 服务 all latency tier without re-embedding**. **关键 NEW**: production embedding model 升级 path 演化为 (a) MRL retrain (new dim 上限) → (b) vector DB 增加更大 dim cell type → (c) 应用层 per-tier prefix dim 选择. **Algorithm 层 cross-model semantic preservation 仍 zero coverage**, 但 within-model multi-tier 已被 MRL 完全解决.

## Answer

### 与之前 ingest 的演进

| | radford-2021-clip post | **kusupati-2022-matryoshka post (NEW)** |
|---|---|---|
| Multi-variant model coexistence (CLIP ViT-B/32 vs ViT-L/14) | **必须重 embed + 重建 index** | **MRL 单 model 取代多 variant: 单 schema + 多 prefix dim 选择** |
| Within-model multi-tier (mobile / cloud / archival) | n/a | **NEW: 单 MRL model 服务 all tier 0-cost truncate** |
| Cross-model migration (BERT → SBERT) | 仍需重 embed | **不变** (MRL 不解决 algorithm 层 cross-model) |

### MRL 取代多 variant 训练范式（NEW frontier closure）

[per kusupati-2022-matryoshka §1 + production reality]

**Pre-MRL pipeline (CLIP-era)**:
- 训练 ResNet-50 (d=512) — mobile deployment
- 训练 ResNet-101 (d=1024) — middle tier
- 训练 ResNet-152 (d=2048) — cloud deployment
- 各档独立训练 + 独立 vector DB schema + 独立 fine-tune pipeline
- 跨档迁移成本极高: 全 corpus 重 embed + 全 index 重建

**Post-MRL pipeline (current production)**:
- 单 MRL model: voyage-3 (1024-d), 内部 nested {8, 16, ..., 1024}
- 单 vector DB schema: store 1024-d
- 应用层 per-tier prefix selection:
  - Mobile / edge: query 取 prefix 256-d → 4× faster ANN
  - Middle tier: prefix 512-d
  - Cloud: full 1024-d
- **0 重 embed, 0 重建 index**

### Vector DB 端 MRL upgrade workflow

[per kusupati-2022-matryoshka + systems/vespa.md + systems/turbopuffer.md]

**MRL major version upgrade (v3 → v4, dim 增大)**:
1. 重新训练 MRL model (v4, e.g., 2048-d, super-set of v3 prefix)
2. Vector DB schema 加新 field (2048-d cell type)
3. Backfill: re-embed corpus with MRL v4
4. Application 切换默认 dim
5. Drop v3 cell type after validation

**Vespa pattern**: application package atomic deploy + multi-tensor field coexistence  
**Turbopuffer pattern**: namespace per MRL version (voyage-3 → voyage-3-v2 → voyage-4)  
**Milvus / Weaviate pattern**: multi-vector field within doc, application select

**MRL minor revision (same dim, fine-tune update)**:
- 仍需 re-embed (different embedding values 但 same schema)
- 但 vector DB schema 不变, deploy 更 lightweight

### MRL 不解决 algorithm 层 cross-model 迁移

[per kusupati-2022-matryoshka §6 + wiki frontier]

MRL 解决: **same model, multi-dim tier** (within-model scaling)
MRL **不解决**: **different model, semantic mapping** (e.g., BERT → SBERT 跨 model)

跨 model migration 仍是 wiki 内**未关闭 frontier**——talk live demo SIGMOD 2026 *Integrating Vector Databases across Embedding Models* 仍是唯一 algorithm-level answer.

### 处理决策表（updated 2026-05-11 post kusupati-2022-matryoshka）

| 场景 | 推荐方案 |
|---|---|
| **Within-MRL-model multi-tier (mobile/cloud/archival)** | **单 schema + 应用层 prefix dim 选择 (0 re-embed)** |
| MRL major upgrade (v3 → v4, dim 增大) | Vespa multi-tensor / Turbopuffer namespace per version / Milvus multi-vector field |
| Cross-model upgrade (CLIP → SigLIP / BERT → SBERT) | 全量重 embed + 全 index 重建 (仍无 algorithm-level 路径) |
| Per-tenant 独立 MRL variant | Turbopuffer namespace-as-tenant |
| Algorithm 层 semantic preserve | 仍未有 production——SIGMOD 2026 |

### 已知盲区

- **MRL prefix v3 vs v4 embedding 几何关系**: 同 model family 不同 fine-tune 之间 prefix 是否互相 compatible? Paper 不验证
- **跨 MRL model prefix compatibility**: voyage-3 256-d prefix vs embed-v4 256-d prefix 是否在 cosine space 接近? 否, 不同 model family. 但 wiki + paper 都 zero coverage 实测
- **Production MRL upgrade frequency**: 多久重 train 一次 MRL? 不公开
- **Algorithm-level cross-model**: 仍是 talk SIGMOD 2026 唯一候选

## Cited Pages

- [concepts/matryoshka-embedding.md](../../concepts/matryoshka-embedding.md)
- [concepts/clip.md](../../concepts/clip.md)
- [topics/adaptive-retrieval-shortlist-rerank.md](../../topics/adaptive-retrieval-shortlist-rerank.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
