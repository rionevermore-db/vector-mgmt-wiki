---
query-key: embedding-update-handling
query: "当 embedding model 升级（比如从 BERT 换到 SBERT，或 OpenAI embedding v2 → v3）时，已有的向量索引和原始文档应该怎么处理？是否必须全量重建？有没有跨模型兼容/复用的方案？"
date: 2026-05-11
phase: post
ingest-context: radford-2021-clip
wiki-pages-total: 71
cited-pages: [concepts/clip.md, topics/multimodal-embedding-retrieval.md, systems/vespa.md, systems/turbopuffer.md, systems/milvus.md]
cited-count: 5
---

# Post-snapshot (radford-2021-clip): embedding-update-handling

## TL;DR (delta from adams-2025-distributedann post)

**CLIP ingest 增强 wiki 的 embedding-side migration 视角**: CLIP 是 wiki 内**第一个明示 embedding model 内部 version variant** 的 source——ViT-B/32 (512-d), ViT-B/16 (512-d), ViT-L/14 (768-d), ViT-L/14@336px (768-d) **4 个 variant 输出维度不同**, 跨 variant migration 直接面对 dim mismatch. **关键 NEW**: production multimodal pipeline 升级 CLIP variant 时 (ViT-B/32 → ViT-L/14) 不仅是 model swap, 还是 **vector dimension change** (512-d → 768-d). 这种 dim 不一致**强制要求 vector DB schema 重建**——不能 in-place upgrade. 各 vendor 的 multi-vector-version coexistence 哲学 (Turbopuffer namespace per model / Vespa multi-tensor field / Milvus multi-vector field) 是 production 处理 CLIP variant upgrade 的关键设计.

## Answer

### 与之前 ingest 的演进

| | adams-2025-distributedann post | **radford-2021-clip post (NEW)** |
|---|---|---|
| Single-model migration tool | Qdrant Aliases + Vespa AppPkg + Turbopuffer copy_from | **不变** |
| Cross-dimension model upgrade case | 仅 BERT-base vs BERT-large 类似 case | **NEW: CLIP ViT-B/32 (512-d) → ViT-L/14 (768-d) 实证 production upgrade pattern** |
| Multi-vector-version coexistence pattern | Vespa multi-tensor + Turbopuffer namespace | **NEW: 在 multimodal context 下 first-class 重要** |
| Algorithm 层跨模型 compat | 仍 zero coverage | **不变** |

### CLIP variant 矩阵: 跨 dimension 升级真实案例（NEW）

[per radford-2021-clip §2.4 Table]

| Variant | Image params | Text params | **Embedding dim** | ImageNet zero-shot |
|---|---|---|---|---|
| RN50 | 25M | 63M | **1024** | ~60% |
| RN101 | 45M | 63M | **512** | ~62% |
| RN50×4 | scale 4× | 63M | **640** | ~67% |
| RN50×16 | scale 16× | 63M | **768** | ~70% |
| RN50×64 | scale 64× | 63M | **1024** | ~74% (12 day × 592 V100) |
| ViT-B/32 | 87M | 63M | **512** | ~63% |
| ViT-B/16 | 87M | 63M | **512** | ~68% |
| ViT-L/14 | 307M | 63M | **768** | ~75% |
| ViT-L/14@336px | 307M | 63M | **768** | **76.2% (paper's "best")** |

→ **Embedding dim 在 4 个不同的值**: 512, 640, 768, 1024. Variant upgrade (e.g., ViT-B/32 → ViT-L/14) 横跨 512 → 768 维度. **Vector DB 端不能 in-place 升级**——必须重建 schema + 重 embed.

### CLIP variant migration 路径（NEW reality）

**Pattern 1: Application package atomic deploy (Vespa)**
```
v1 schema: tensor<float>(x[512])  # ViT-B/32
v2 schema: tensor<float>(x[768])  # ViT-L/14

# Vespa 通过 application package 切换:
# 1. Deploy v2 schema (新 field 共存 v1)
# 2. Backfill: 重 embed 所有 docs with ViT-L/14
# 3. Query 切换: ranking profile 引用 v2 field
# 4. Drop v1 field via 下次 deploy
```

**Pattern 2: Namespace-as-version (Turbopuffer)**
```
v1 namespace: "products-clip-vitb32"  # 512-d
v2 namespace: "products-clip-vitl14"  # 768-d

# 渐进 migration:
# 1. 创建 v2 namespace, schema 自动推断
# 2. 重 embed 写入 v2
# 3. Application 切换 namespace path (atomic alias)
# 4. Drop v1 namespace
```

**Pattern 3: Multi-vector field within doc (Milvus / Weaviate)**
```
# Milvus collection 加 v2 vector field 共存:
fields: [
  vector_v1: float_vector(dim=512),  # ViT-B/32
  vector_v2: float_vector(dim=768),  # ViT-L/14
]
# Backfill v2, query 切换索引选择
```

→ **3 种 production multimodal-version-migration pattern 完整覆盖 wiki vendor**: Vespa AppPkg / Turbopuffer namespace / Milvus multi-field. 都 work, 选择基于 deployment 哲学.

### 处理决策表（updated 2026-05-11 post radford-2021-clip）

| 场景 | 推荐方案 |
|---|---|
| **Multimodal CLIP variant upgrade (跨 dim)** | **Vespa application package** OR **Turbopuffer namespace per version** OR **Milvus multi-vector field** |
| Per-tenant 独立 multimodal model | **Turbopuffer namespace-as-tenant** |
| 同 doc 多 model embedding 共存 | **Vespa multi-tensor field** OR **Weaviate named vectors** |
| Cross-modal CLIP fine-tune (BiomedCLIP / RemoteCLIP) production routing | **Turbopuffer per-domain namespace** OR **Vespa rank-profile dispatch** |
| **Algorithm 层跨 CLIP-variant semantic preserve** | 仍未有 production——等 SIGMOD 2026 cross-model alignment |

### CLIP-related embedding migration frontier（NEW）

[per radford-2021-clip §6 limitations + production reality]

1. **CLIP fine-tune migration**: production system 常 fine-tune CLIP on domain data (OpenCLIP variants). Fine-tune 后 embedding 距离 base CLIP 多远? Vector DB index 是否需要 rebuild 还是 partial reindex? **wiki + 论文 zero coverage**
2. **CLIP → SigLIP migration**: SigLIP 与 CLIP 同 dim (768-d) 但 loss 不同——embedding space 几何关系? 是否可 direct swap? **未知**
3. **CLIP → ImageBind**: 1024-d 6 modality, vs CLIP 768-d 2 modality. 跨 modality count upgrade 是否 break index? **未知**
4. **Period-of-time fine-tune (CLIP every 6 months retrain)**: 持续 embedding drift, vector DB rebuild 周期与频率? **production case 不公开**

### Algorithm 层跨模型方案——frontier 仍未关闭

CLIP paper §6 不讨论 algorithm-level cross-model compatibility. **Talk 当天 SIGMOD 2026 live demo 仍是唯一 algorithm-level answer**——CLIP variant migration 在 vector DB 端目前**全部走 schema-level / namespace-level 重建路径**, 不存在 semantic-preserving mapping.

### 已知盲区

- **CLIP-style embedding 跨 model semantic stability**: ViT-B/32 vs ViT-L/14 输出在 cosine space 是否有可学习的 mapping? Probably 否, 但 wiki / 论文 zero coverage
- **CLIP fine-tune 后 ANN index reuse**: 部分 fine-tune (e.g., 仅 text encoder fine-tune) 是否能保留 image-side ANN index? 未知
- **CLIP vs SigLIP embedding cross-compat**: 同 dim 同 cosine space, 是否实测可互换? 不存在公开数据
- **Multi-version production routing cost**: Vespa multi-tensor / Turbopuffer multi-namespace / Milvus multi-field 各 vendor production 实际运维成本 + recall guarantee 量化数据未公开

## Cited Pages

- [concepts/clip.md](../../concepts/clip.md)
- [topics/multimodal-embedding-retrieval.md](../../topics/multimodal-embedding-retrieval.md)
- [systems/vespa.md](../../systems/vespa.md)
- [systems/turbopuffer.md](../../systems/turbopuffer.md)
- [systems/milvus.md](../../systems/milvus.md)
